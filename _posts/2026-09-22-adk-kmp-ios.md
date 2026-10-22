---
layout: post
title: "ADK for Kotlin 支持 iOS 吗？我拆开它的 Gradle 元数据查了一遍"
date: 2026-09-22 07:11:05 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/adk-kmp-ios/cover.png
---

![ADK for Kotlin 支持 iOS 吗？我拆开它的 Gradle 元数据查了一遍](/assets/images/adk-kmp-ios/cover.png)

## 先认个错

之前介绍 ADK for Kotlin 1.0 的时候，我写过一句"未来如果 iOS 侧也支持，那就是一套代码三端复用"。当时这个词用得很滑——它跟"KMP 核心"这句话摆在一起，很容易让人读成"iOS 应该也能跑"。

后来有读者来问：它既然是 Kotlin Multiplatform 写的，那 iOS 是不是本来就能用？

这个问题我当初不该带过。所以这次我自己去查了，查的方式不是看文档怎么说，而是直接拆它的构建产物。

结论先放这儿：**不能。而且这不是加一行 target 就能解决的事。**

## 查一个 KMP 库到底支持什么，有个很硬的地方

先说方法，因为这个方法比这次结论更有用。

KMP 库在发布时会生成一份 **Gradle module metadata**（`.module` 文件），里面列着这个库实际编译出来的所有平台变体。它是构建系统的产物，不是谁写的宣传文案——一个库说自己"支持多平台"，但元数据里没有对应的变体，那构建时就是拿不到，编译直接失败。

所以想知道一个库的真实平台覆盖，一行命令就够：

```bash
# 通用查法：把 group 换成包名路径
curl -sL "https://repo1.maven.org/maven2/com/google/adk/google-adk-kotlin-core/1.1.0/google-adk-kotlin-core-1.1.0.module" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
vs = d.get('variants', [])
plats = sorted({v['attributes'].get('org.jetbrains.kotlin.platform.type', '?') for v in vs})
names = [v['name'] for v in vs]
ios = [n for n in names if 'ios' in n.lower()]
print('platform.type :', ', '.join(plats))
print('含 ios 的变体 :', len(ios), '个')
for n in ios[:4]: print('   -', n)
"
```

拿 ADK for Kotlin 跑一下，输出是：

```
platform.type : androidJvm, common, jvm
含 ios 的变体 : 0 个
```

## 怎么读这个结果

有个地方容易看错，我自己第一次就差点搞混：**iOS target 不会出现在 `platform.type` 里。**

Kotlin/Native 在 `platform.type` 上统一标记为 `native`，具体是哪个 Apple 平台，看的是**变体名**。所以正确的读法是两层一起看：

- `platform.type` 有 `native` → 说明这个库编译了 Kotlin/Native 目标
- **变体名里出现 `iosArm64` / `iosSimulatorArm64` / `iosX64`** → iOS 才是真能用

拿一个确定支持 iOS 的库对照一下，比如 Ktor：

```
platform.type : common, js, jvm, native, wasm
含 ios 的变体 : 9 个
   - iosArm64ApiElements-published
   - iosArm64SourcesElements-published
   - iosSimulatorArm64ApiElements-published
   ...

```

64 个变体，9 个带 ios。这才是"真支持 iOS"的样子。

ADK 那边是 8 个变体，`platform.type` 只有 androidJvm / common / jvm，**带 ios 的变体是 0 个**。

我顺手验了一遍所有版本，包括最新的 1.1.0：都没有 iOS。

再看源码结构，结论是一致的。`core/src/` 下面的 source set 是这些：

```
androidMain
commonJvmAndroidMain      ← 注意这个名字
commonMain
jvmMain
（没有 iosMain）
```

而 `core/build.gradle.kts` 里的 target 声明，只有 `jvm()` 加 Android，一行 iOS 都没有。

Maven Central 上 `com.google.adk` 这个 group 下，**iOS 相关的 artifact 是 0 个。**

## 三道坎，一道比一道难

知道"没有 iOS"还不够，值得看看为什么没有。因为答案决定了"自己动手"要花多大代价。

### 坎一：他们特意劈了一层叫 `commonJvmAndroidMain`

这个 source set 名字本身就是答案——JVM 和 Android 共享的那部分。它 `dependsOn(commonMain)`，里面装着：

| 依赖 | iOS 上的状况 |
|---|---|
| `kxml2`（XML 解析） | 没有 Kotlin/Native 版本，得换库或写 actual |
| `snakeyaml`（YAML） | 同上 |
| `opentelemetry-api`（Java 版遥测） | iOS 要接 OpenTelemetry 的 Swift 实现，整层得做成可插拔 |
| `reactive-streams` | **JVM 独有规范，Apple 平台根本没有对应物** |

那条 `reactive-streams` 是关键信号。它是 JVM 世界的响应式流规范，苹果那边不存在这个东西。而 ADK 的 Java 互操作签名就建立在这上面——源码注释里的原话是 "the api-vs-implementation question for the Java-interop signatures"。

作者其实很清楚代码分两层：一层是真正的跨平台公共逻辑，一层是"JVM 和 Android 共享"的逻辑。**他们特意做了这个区分，就是因为它现在只想支持这两个平台。**

### 坎二：`jvmMain` 里也全是 JVM 专属

`gson`、`google-cloud-storage`、`mcp`（MCP 的 Java SDK）、`ktor`。Google Cloud Storage 和 MCP 的 Java SDK 都没有 iOS 版本，会话持久化和云存储那部分要另给 iOS 写一套。

### 坎三（这条最要命）：`google-genai-kotlin` 自己也不支持 iOS

`commonMain` 的依赖里有这么一行：

```kotlin
commonMain {
  api(libs.google.genai.kotlin)   // com.google.genai:google-genai-kotlin:1.0.0
}
```

我拿同一个方法查了它：

```
平台变体: androidJvm, common, jvm
```

**Google 自家那个生成式 AI 的 Kotlin SDK，也没有 iOS target。**

而且它是 `api` 依赖，不是 `implementation`——意思是它出现在所有消费者的编译路径上，绕不过去。

所以真相是：**想让 ADK for Kotlin 跑 iOS，你得先把 Google 的 GenAI Kotlin SDK 移植到 iOS。** 那是另一个 SDK 的工程量，而且东西在 Google 手里，不在你能改的范围内。

## 那"自己编译支持 iOS"到底意味着什么

不是 fork 一下加三行 target。粗略拆一下：

| 要做的事 | 量级 |
|---|---|
| 补 iOS target + 各层 actual 实现（XML、YAML、遥测、持久化、网络） | 中到大 |
| `reactive-streams` 换成 Flow，重设计 API（会破坏 Java 互操作承诺） | 中 |
| 移植 google-genai-kotlin 到 iOS，或绕开它直连 REST | **大** |

后两项都不在你的仓库里。尤其第三项——那是要往上动别人的东西。

## 1.1.0 的方向，反而把这件事说清楚了

ADK for Kotlin 1.1.0 是 9 月 15 日发的（1.0 是 9 月 7 日 GA，1.0.1 是 9 月 10 日，节奏很快）。看它的新增特性：

- `AppFunctionsToolset`，对接 Android 的 AppFunctions
- Android AppFunctions 转换层
- 把 app 的 AppFunctions 说明注入模型指令
- 针对 Java 的 `BaseFutureSkillSource` 基类、长时运行工具的 Java 版示例
- `Plugin.onRunError` 回调

修的问题也在同一片区域：未知工具名不再终止整个调用、workflow agent 的会话恢复修正、MCP server 拒绝请求时保持 turn 存活。

注意这些功能的重心。1.1.0 里最实在的新能力是 **Android AppFunctions** 那条线——让 agent 去调用 Android 应用暴露的函数。

这说明 iOS 的缺口不是疏忽，是**战略选择**。Google 的精力在 Android 侧：Gemini Nano、ML Kit、LiteRT-LM、Firebase AI、现在的 AppFunctions，全是为 Android 铺的。ADK for Kotlin 的 "Kotlin Multiplatform" 目前是 "Kotlin Multiplatform（Android + JVM）"的缩写——那个 common 层是给这两个平台共享用的，不是给 iOS 预留的。

## 那做 iOS 的人怎么办

三条路，按我的推荐顺序。

**路线 A：Agent 放服务端，iOS 只做客户端（推荐）**

ADK for Kotlin 的 **JVM 那条腿是完整可用的**——这是它的主战场之一，不是残废版。

所以把 agent 逻辑跑在服务端 JVM 上，iOS 通过 HTTP 调过去。而 ADK 生态里正好有为此准备的东西：A2A（Agent-to-Agent）协议，`google-adk-kotlin-a2a` 这个模块就是干这个的（它同样只有 JVM 版，但服务端本来就用 JVM）。

代价是失去端侧能力——隐私、离线、低延迟这三样没法兼顾。
好处是今天就能用，零移植风险。

对大多数产品来说这个取舍是划算的：你的 agent 需要联网调模型，那它本来就不是纯离线的。

**路线 B：自己写一层"ADK-lite for iOS"**

只取 commonMain 的设计思路（编排、会话、记忆、工具 schema），在 iOS 侧用 KMP 生态里现成的积木重搭：

- Ktor 做网络（KMP 原生支持 iOS）
- 直接调 Gemini REST API，绕开 `google-genai-kotlin`
- SQLDelight 做持久化

但这就不是"用 ADK"了，是照着它的设计自己实现一遍。业务逻辑能共享，框架得重写。

**路线 C：等，或者催**

1.0 到 1.1.0 只隔了 8 天，迭代速度不慢。如果 iOS 在路线图上，等是划算的。

顺便说一件我顺手做的事：我在仓库里开了个 issue 把这件事问清楚了——

> https://github.com/google/adk-kotlin/issues/504

问了三件事：iOS 在不在路线图上、真正的阻塞点是什么（我怀疑是 GenAI Kotlin SDK 缺 native target，而不是 ADK 本身）、现阶段官方推荐什么架构。

之所以觉得这个 issue 有价值：这个仓库目前总共只有几个 issue，而其中一个上个月提的 feature request 两天内就被打了 `need attention` 标签。说明团队确实在看。有 iOS 需求的同学可以去点个赞，需求信号比什么都管用。

## 比结论更值钱的是那个查法

回到开头那个方法。

以后你评估任何 KMP 库——不管是 ADK、还是别的什么"跨平台"框架——别只看 README 和发布说明。README 会写 "runs on all supported platforms" 这种话，看着很全，但 "supported" 是谁定义的、支持到什么程度，文档通常不说。

**去看 `.module` 文件。** 那里面列的是构建系统真正能拿到的东西：

- 有 `iosArm64` / `iosSimulatorArm64` → iOS 真能用
- 有 `js` / `wasmJs` → Web 真能用
- 只有 `common` / `jvm` / `androidJvm` → 那就是只有 JVM 和安卓

一行 curl 就能做完的事，能省掉你两周的重构。

KMP 这两年最大的认知陷阱就是这个：**"基于 KMP 构建"和"KMP 全平台可用"是两件不同的事。** 前者说的是代码怎么组织的，后者说的才是你能拿到什么。ADK for Kotlin 是个很好的例子——它的内核确实是精心设计过的跨平台架构，但它目前选择把 iOS 那扇门关着。

---

*核查方式：直接读取 `com.google.adk:google-adk-kotlin-core` 各版本（含 1.1.0）在 Maven Central 上的 Gradle module metadata、`core/build.gradle.kts` 与 `core/src/` 的 source set 结构、`com.google.genai:google-genai-kotlin` 的平台变体，并核对了 ADK for Kotlin v1.0.0 / v1.0.1 / v1.1.0 的 Release Notes。*
