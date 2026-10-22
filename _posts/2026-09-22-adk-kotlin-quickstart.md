---
layout: post
title: "ADK for Kotlin 1.1 + Ktor 3.6：给 KMP 开发者的 AI agent 上手指南"
date: 2026-09-22 07:55:03 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/adk-kotlin-quickstart/cover.png
---

![ADK for Kotlin 1.1 + Ktor 3.6：给 KMP 开发者的 AI agent 上手指南](/assets/images/adk-kotlin-quickstart/cover.png)

Python 那边的 agent 框架已经换过好几轮了，LangChain、LlamaIndex，再到 Google 自己出的 ADK。Kotlin 这边一直没什么动静——不是没人想做，是缺一个官方托底的东西，大家不太敢往生产里放。

Google 的 `adk-kotlin` 1.0 在 9 月 7 号 GA，1.1.0 是 9 月 15 号，Ktor 3.6.0 的构件 9 月 16 号上的 Maven Central。我拿这三个搭了个能用的东西：**一个能自己查证「某个 KMP 库到底支不支持 iOS」的 agent**，服务端和客户端放在同一个 KMP 工程里。

先交代一件事。写这篇文章的过程中我踩了个坑，照着源码写的请求体在真服务器上直接返回 400，而且**不带任何错误信息**。折腾了小半个钟头才定位到，是版本问题。这个放在后面单独讲，因为它比教程本身更值得记住。

## 这个 agent 解决什么问题

KMP 开发者最常问的一个问题是：「这个库支持 iOS 吗？」

README 经常不给答案，或者给的是过期的答案。真正权威的地方是库发布到 Maven Central 上的 **Gradle module metadata**——那个 `.module` 文件是构建过程生成的，里面列的就是这次编译真实产出的平台变体。

所以工具做的事很直接：把 `.module` 拉下来，解析出所有变体名，数一数里面有没有 iOS 相关的。

```kotlin
@Tool
fun checkLibraryPlatforms(
  @Param("Maven group id, e.g. com.google.adk") group: String,
  @Param("Maven artifact id, e.g. google-adk-kotlin-core") artifact: String,
  @Param("Version to check, e.g. 1.1.0") version: String,
): String {
  val url = "https://repo1.maven.org/maven2/${group.replace('.', '/')}/" +
    "$artifact/$version/$artifact-$version.module"
  // ... 拉取、解析、统计 iOS 变体
}
```

`@Tool` 和 `@Param` 是 ADK 的注解，后面会说它为什么重要。

## 结构：为什么是两个模块

```
adk-kotlin-agent-quickstart/
├── shared/          KMP 模块
│   └── src/
│       ├── commonMain/   Ktor 客户端 + 协议 DTO（jvm / iOS 共用）
│       ├── jvmMain/      CLI 入口 + CIO 引擎
│       └── iosMain/      Darwin 引擎
└── agent-server/    纯 JVM 模块：ADK agent + HTTP 服务
```

看起来可以塞进一个模块，但塞不进去。`google-adk-kotlin-webserver` 内部依赖的是 **Ktor 2.3.13**，而客户端要用 **Ktor 3.6.0**。放一起的话 Gradle 按「最高版本获胜」把服务端也拉到 Ktor 3，而 ADK 的代码是按 Ktor 2 的 API 编译的，运行时会直接炸。

这不是猜的。跑起来之后看类路径，服务端确实是这一串：

```
ktor-server-netty-jvm-2.3.13.jar
ktor-server-core-jvm-2.3.13.jar
ktor-client-core-jvm-2.3.13.jar
...
```

客户端那边是 3.6.0。物理隔开，各自安好。

## 版本锁定

KSP 必须和 Kotlin 严格配对，这是最容易埋雷的地方。我直接照抄了 `google/adk-kotlin` 仓库自己构建用的组合：

```toml
[versions]
kotlin = "2.3.21"
ksp = "2.3.9"          # 和 kotlin 一起升，别单独动
adk = "1.1.0"
ktor = "3.6.0"
```

服务端依赖三个：

```kotlin
dependencies {
  implementation(libs.adk.core)        // agents / models / tools / sessions / runners
  ksp(libs.adk.processor)              // 编译期生成工具 schema
  implementation(libs.adk.webserver)   // 把 agent 用 HTTP 暴露出去
}
```

## 工具是怎么变成 schema 的

ADK for Kotlin 和 Python 版最大的差别在这儿。Python 靠运行时反射，Kotlin 这边是 KSP **编译期**生成。

`KmpSupportAgent` 里的这一行是关键：

```kotlin
tools = KmpLibraryService().generatedTools()
```

`generatedTools()` 这个函数在整个工程里我一行都没写过。编译完之后，它出现在：

```
agent-server/build/generated/ksp/main/kotlin/com/example/adk/server/
├── CheckLibraryPlatformsTool.kt
├── ListAvailableVersionsTool.kt
└── KmpLibraryService_GeneratedTools.kt
```

后者的内容是：

```kotlin
public fun KmpLibraryService.generatedTools(): List<FunctionTool> = listOf( ... )
```

好处是运行时不需要反射，R8/混淆、Native 镜像、启动时间这几个方向上都干净。代价是工具签名改了必须重新编译，动态注册工具这种玩法就别想了。

## agent 本体

```kotlin
object KmpSupportAgent {
  @JvmField
  val rootAgent = LlmAgent(
    name = "kmp_library_platform_agent",
    model = Gemini(name = "gemini-3.1-flash-lite"),
    instruction = Instruction("""
      You are an assistant for Kotlin Multiplatform developers.
      When the user asks whether a library supports a platform, never answer
      from memory. Call `checkLibraryPlatforms` and read the result.
      ...
    """.trimIndent()),
    tools = KmpLibraryService().generatedTools(),
  )
}
```

顺带说下 `@JvmField`。官方示例里每个 agent 的 `rootAgent` 都带着它，我一开始也照抄了，还以为它是必需的。其实它只是让 Kotlin `object` 的属性变成真正的静态字段（而不是 `INSTANCE.getRootAgent()`），方便 Java 调用方访问。我试着去掉它重新编译运行，服务端照样起来、`/list-apps` 照样返回——因为 `AdkServerConfig.inMemory(agent)` 是直接拿 agent 对象，全程不走反射。抄没问题，但别以为少了它就启动不了。

instruction 里我特意写清楚了一件事：`platform.type` 是个聚合标记，Kotlin/Native 的目标统一报 `native`，具体是哪个 Apple 目标只出现在**变体名**里。不写这句，模型容易看着 `native` 这个词就自己下结论，而它并不能从 `native` 推出到底有没有 iOS。

## 起服务

```kotlin
fun main() {
  val config = AdkServerConfig.inMemory(KmpSupportAgent.rootAgent)
  AdkDevServer(config).start(wait = true)
}
```

`AdkDevServer` 会顺带挂上开发界面。路由长这样：

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/list-apps` | 有哪些 agent，也就是 `appName` 能填什么 |
| GET | `/version` | 服务端版本 |
| POST | `/apps/{appName}/users/{userId}/sessions` | 建会话，返回会话 id |
| POST | `/run` | 跑一轮，一次性返回完整事件列表 |
| POST | `/run_sse` | 跑一轮，SSE 流式返回 |
| GET | `/dev-ui` | 开发界面 |

实测三个能直接验：

```
$ curl -s localhost:8080/list-apps
["kmp_library_platform_agent"]

$ curl -s localhost:8080/version
{"version":"1.1.0","language":"kotlin","language_version":"21.0.10"}

$ curl -s -X POST -H "Content-Type: application/json" -d '{}' \
    localhost:8080/apps/kmp_library_platform_agent/users/demo-user/sessions
{"id":"b86f7324-81db-49df-8007-1700077ebdb4","appName":"kmp_library_platform_agent",...}
```

`/run_sse` 用的是标准 SSE，每个事件一行 `data: <json>`，后面跟一个空行。服务端的实现是手写的，因为 Ktor 2.x 没有 SSE 插件。源码里的注释挺有意思：

> Ktor 2.x has no SSE plugin, so stream manually. Use async respondBytesWriter, not respondTextWriter, whose blocking bridge parks via a coroutines internal gone in 1.11.0.

> Ktor 2.x 没有 SSE 插件，所以手动流式输出。要用异步的 `respondBytesWriter`，不能用 `respondTextWriter`——它的阻塞桥接依赖了一个在 kotlinx-coroutines 1.11.0 里已经被删掉的内部实现。

也就是说这一句注释本身在提醒你：升级 coroutines 版本可能把这个 API 弄坏。

## 客户端：写在 commonMain 里

真正跨平台的部分在这里。请求构造和 SSE 解析全部写在 `commonMain`，不碰平台 API：

```kotlin
fun run(prompt: String, sessionId: String? = null): Flow<Event> = channelFlow {
  val sid = sessionId ?: createSession()
  httpClient.preparePost("$baseUrl/run_sse") {
    contentType(ContentType.Application.Json)
    setBody(RunRequest(appName, userId, sid, Content.text("user", prompt), streaming = true))
  }.execute { response ->
    val channel = response.bodyAsChannel()
    while (true) {
      val line = channel.readLine() ?: break
      if (!line.startsWith("data:")) continue
      val payload = line.removePrefix("data:").trim()
      if (payload.isEmpty()) continue
      val event = json.decodeFromString<Event>(payload)
      send(event)
      if (event.turnComplete) break
    }
  }
}
```

只有引擎是分平台的，用一个 `expect/actual` 收口：

```kotlin
// commonMain
expect fun createPlatformHttpClient(json: Json): HttpClient

// jvmMain
actual fun createPlatformHttpClient(json: Json) = HttpClient(CIO) { ... }

// iosMain
actual fun createPlatformHttpClient(json: Json) = HttpClient(Darwin) { ... }
```

循环退出靠两个字段：`partial` 标记这是一个还没说完的流式分片，`turnComplete` 标记这一轮结束。没有 `turnComplete` 就得靠连接关闭来判断，那会拖到超时。

## 那个 400

现在讲开头说的坑。

我照着自己的理解写请求体，用了 `app_name` / `user_id` / `new_message` 这套命名——毕竟 Python 版 ADK 就是这样，而且我确实在仓库源码里看到 `ApiModels.kt` 给每个多词属性都挂了一个别名：

```kotlin
@JsonNames("app_name") val appName: String,
@JsonNames("user_id") val userId: String,
```

请求发出去：

```
HTTP/1.1 400 Bad Request
Content-Length: 0
```

没有 body，没有堆栈，服务端日志里连一行 ERROR 都没有。试着换成 camelCase 再发一次，返回变成 **500**——这说明反序列化通过了，失败发生在更后面（模型调用）。也就是说：

- camelCase → 解析成功，一路走到 Gemini SDK
- snake_case → 根本没进到业务逻辑

`@JsonNames` 在 kotlinx-serialization 里是「备用名」，理论上两种拼写都应该认。我甚至单独写了个最小复现，用同样的注解和同样的 Json 配置，snake_case 解析是**成功**的。所以问题不在 kotlinx。

问题在版本。我去翻了 tag：

```
v1.1.0 的 ApiModels.kt：
  data class AgentRunRequest(
    val appName: String,
    val userId: String,
    ...
  )
  → 一个 @JsonNames 都没有
```

那两个别名提交 `94dcdd13`（`feat: accept snake_case and camelCase in web server request bodies`）的日期是 **2026-09-15**，进的是 `main`，**在 `v1.1.0` tag 之后**。仓库里还有个测试文件写着「服务端读 snake_case」——它描述的是 main 的行为，不是你能下载到的 1.1.0 的行为。

拿真实的 `AgentRunRequest` 和真实的 `adkJson` 解码 snake_case，报错很明确：

```
kotlinx.serialization.MissingFieldException: Fields [appName, userId] are required
for type with serial name '...AgentRunRequest', but they were missing
```

所以规则是：**对 1.1.0，一律用 camelCase。** snake_case 要等下一个版本。

这件事的教训不是「要看 tag」，而是「读 main 的源码不等于读你依赖的那个版本」。我如果没真的把请求发出去试一遍，这篇文章里的示例代码就是照抄一个未发布行为、一跑就 400 的代码。

真要说自查手法，`/version` 这个端点很有用——服务端会自报版本，然后去对照那个 tag 的源码，比对着 main 猜要靠谱。

## 工具跑出来的样子

这一段不需要模型，工具是纯 Kotlin，可以直接跑：

```bash
./gradlew :agent-server:runTools
```

先拿 ADK 自己开刀：

```
com.google.adk:google-adk-kotlin-core:1.1.0
platform.type values: androidJvm, common, jvm
variants mentioning iOS: 0
Verdict: no iOS target. This library cannot be consumed from an
iosArm64 / iosSimulatorArm64 source set.
```

再拿 Ktor 对照：

```
io.ktor:ktor-client-core:3.6.0
platform.type values: common, js, jvm, native, wasm
variants mentioning iOS: 9
  - iosArm64ApiElements-published
  - iosSimulatorArm64ApiElements-published
  - iosX64ApiElements-published
  ...（共 9 个，含 Api / Metadata / Sources 三类）
Verdict: iOS targets are published, so Apple targets can depend on it.
```

同一个函数、同一套判断逻辑，两个库的结论正好相反。这个反差下面单独说。

版本查询和错误路径也顺带看一眼：

```
com.google.adk:google-adk-kotlin-core — latest=1.1.0
recent: 0.5.0, 0.6.0, 0.7.0, 0.8.0, 0.9.0, 1.0.0, 1.0.1, 1.1.0

Could not fetch metadata for com.google.adk:does-not-exist:9.9.9 (HTTP 404).
The coordinates or version are probably wrong.
```

错误路径返回的是一句人话，不是抛异常。工具返回给模型的是字符串，异常直接把这一轮对话打断，模型连「这个库不存在」都说不出来。

## 跨端这件事

`shared` 这块是真跨平台。我实际编译过：

```bash
$ ./gradlew :shared:compileKotlinIosSimulatorArm64 :shared:compileKotlinIosArm64
BUILD SUCCESSFUL in 33s
```

同一份 `commonMain` 代码，JVM 和 iOS 都过。

而 `agent-server` 永远只能在 JVM（或者 Android）上。这不是配置问题——`google-adk-kotlin-core` 发布到 Maven Central 的 Gradle 变体只有 `common` / `androidJvm` / `jvm` 三个，`google-genai-kotlin` 也一样。**没有 iOS 变体，也没有 Native 变体。**

所以如果你想在 iOS 上跑 agent，路只有两条：把它放在服务端，iOS 只做客户端（就像这个工程的结构）；或者等上游补 Native 支持。我为此在 ADK 仓库开了个 issue，编号 [#504](https://github.com/google/adk-kotlin/issues/504)，目前还没回复。

顺带一说，这个「agent 只在服务端」的切分其实多数场景更合理——模型密钥不会进客户端包，工具执行和权限检查也都在你能控住的地方。

## 怎么跑

```bash
export GOOGLE_API_KEY=...
./run-stack.sh
```

这个脚本干四件事：把服务端装出来并起在 8080、轮询 `/list-apps` 等它真就绪、跑客户端、跑完收掉服务端并把客户端的退出码原样传出来。

（第一版这个脚本是漏的：它借 Gradle 生成的启动脚本起服务，而那个脚本把 java 当子进程跑，`kill` 只杀掉外壳，服务端会变成孤儿一直占着 8080。改成直接起 java，`$!` 才是服务端本体。）

也可以拆开：

```bash
./gradlew :agent-server:run     # 终端 1
./gradlew :shared:runClient     # 终端 2
```

最后交代清楚我验证到哪一步了，别让你照着跑的时候产生落差。

跑通的：整个工程编译通过（含 KSP 代码生成）；服务端启动；`/list-apps`、`/version`、建会话都返回 200；客户端发出的 `/run_sse` 请求被服务端接受，并进入 `com.google.genai.kotlin` 的模型调用层。

没跑通的：**一次真实的模型对话**。这需要你自己的 Gemini API key，我这边没有。所以文章开头那个「查 iOS 支持情况」的完整对话输出，我没有贴——它得接上有效 key 才能看到，我不想贴一段没跑过的输出。

代码在 https://github.com/trewvip-arch/adk-kotlin-agent-quickstart

Apache-2.0。有问题直接在仓库里开 issue。
