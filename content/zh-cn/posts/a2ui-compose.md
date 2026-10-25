---
title: "A2UI 实测：agent 画界面，Jetpack Compose 渲染"
date: 2026-09-30T09:15:53+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "AI Agent"]
summary: "同一份 JSON，同一个 App，同一次运行。"
cover:
    image: "/images/a2ui-compose/cover.png"
    alt: "A2UI 实测：agent 画界面，Jetpack Compose 渲染"
    relative: false
---

## 先看两张截图

同一份 JSON，同一个 App，同一次运行。

第一张，表单是满的：访客 Ada Lovelace、日期 Oct 1, 2026、时间 7:30 PM、人数 2，下面两个按钮 Cancel 和 Confirm booking，全套 Material 3 的原生控件。

![字段填充正常](/images/a2ui-compose/fields-filled.png)

第二张，同一个表单，字段全空。不光空，两个日期时间控件的标签都没了，只剩两个灰色圆角空框。

![同一份 JSON，字段全空](/images/a2ui-compose/fields-empty.png)

差别只在于我什么时候把 `updateDataModel` 这条消息发进去。前一秒和组件一起发，字段就丢；晚一秒单独发，字段就落上。这条我后面细说，它是我这次踩到的最值钱的一个坑。

## A2UI 是什么

一句话，官方原话：

> A2UI (Agent to UI) is a declarative UI protocol for agent-driven interfaces. AI agents generate rich, interactive UIs that render natively across platforms (web, mobile, desktop) without executing arbitrary code.

agent 不吐 HTML 或者 JavaScript，它吐一段声明式 JSON 描述界面长什么样，客户端拿自己的原生组件把它渲染出来。官方给的对照场景特别典型：

> User: "Book a table for 2 tomorrow at 7pm"
> Agent: "Okay, for what day?"
> User: "Tomorrow"
> Agent: "What time?"

来回三轮只为填一个表单。换成 agent 直接生成一个带日期选择器、时间选择器和提交按钮的表单，用户是在操作界面，不是在跟它说话。

这套协议里我觉得最聪明的设计是 catalog（组件目录）。它不是让 agent 自由发明界面，而是先由客户端声明一份契约：我可以用这些组件、这些属性、这些本地函数。agent 只能在契约里挑选和组合。协议原文写得很直白：

> Rather than having an agent generate arbitrary UI code or invent unregistered components, the catalog acts as a contract that defines the specific UI elements, properties, and functions available to the agent.

加上「传的是数据不是代码」这条，安全上的账就好算了：没有 iframe，也没有任意代码执行。

## Compose 这边的官方渲染器

9 月 23 日 AndroidX 发了 Jetpack Compose 的 A2UI 渲染器，1.0.0-alpha01。我在 Google Maven 上逐个确认过这些 artifact 都存在：

```kotlin
implementation("androidx.a2ui.compose:compose-runtime:1.0.0-alpha01")  // 基于 Compose Snapshot 状态做细粒度重组
implementation("androidx.a2ui.compose:compose-ui:1.0.0-alpha01")       // 组件与 catalog 定义
implementation("androidx.a2ui:a2ui-model:1.0.0-alpha01")               // 协议数据层
implementation("androidx.a2ui:a2ui-engine:1.0.0-alpha01")              // 解析、schema 校验
implementation("androidx.compose.material3:material3-a2ui:1.0.0-alpha01")  // Basic Catalog 的 Material 3 实现
```

`material3-a2ui` 就是成品：Basic Catalog 的 Material 3 实现，一行就能起。你要给它补的是图像、视频、音频播放器，以及打开 URL 和处理本地化文案的回调。官方故意不带 Coil 和 Media3：

> The A2UI libraries intentionally don't bundle external media and networking dependencies, such as Coil, Glide, or Media3. Instead, you provide your own renderers.

还有一个 KMP 开发者会立刻关心的问题。我拆了 `compose-runtime` 的 Gradle module 元数据：

```text
- releaseVariantReleaseApiPublication     type=aar
- releaseVariantReleaseRuntimePublication type=aar
```

只有 aar，没有 klib 也没有 JVM 变体。这一版是 Android 独占，Compose Multiplatform 现在用不上。

## 最小接线

数据层就两个对象：一个 parser，一个 processor。

```kotlin
private val parser = A2uiMessageParser()

private val catalog = materialA2uiBasicCatalogV1(
    image = MaterialA2uiBasicCatalogV1Defaults.image { url, _, _, modifier, _ ->
        Placeholder("image", url, modifier)
    },
    urlOpener = { url -> Log.i(TAG, "open $url") },
    messageFormatter = { pattern, locale, args -> /* MessageFormat */ },
    localeProvider = A2uiLocaleProvider.Default,
)

private val processor = A2uiMessageProcessor(catalogs = listOf(catalog))
val surfaces = processor.activeSurfaces

init {
    viewModelScope.launch(Dispatchers.Default) { processor.collectMessages() }
    viewModelScope.launch(Dispatchers.Default) {
        processor.outboundEvents.collect { Log.e(TAG, "OUTBOUND: $it") }
    }
}
```

界面侧就是一遍遍历，把每个 surface 交给 `A2uiSurface`：

```kotlin
surfaces.forEach { surface ->
    key(surface.id) {
        A2uiSurface(surfaceModel = surface, modifier = Modifier.fillMaxWidth())
    }
}
```

喂进去的是三段协议消息，`createSurface` 开画布、`updateComponents` 描述组件树、`updateDataModel` 给绑定值：

```json
{ "version": "v0.9",
  "createSurface": { "surfaceId": "booking",
                     "catalogId": "https://a2ui.org/specification/v0_9/catalogs/basic/catalog.json" } }

{ "version": "v0.9",
  "updateComponents": { "surfaceId": "booking",
    "components": [
      { "id": "root", "component": "Column", "children": ["title", "nameField", "actions"] },
      { "id": "title", "component": "Text", "text": "Book a Table", "variant": "h1" },
      { "id": "nameField", "component": "TextField", "label": "Guest name",
        "value": { "path": "/booking/name" }, "variant": "shortText" }
    ] } }
```

注意组件属性里的 `value` 不是值本身，而是一个 `path`，指向数据模型。这也是为什么第三段消息发不发得进去，决定你的表单是满的还是空的。

## 跑起来之后，坑一个接一个

下面每一条都是我真撞上、并且找到原因或绕法的，官方那两页文档一条都没写。

一、`updateDataModel` 跟 `updateComponents` 同一批发，会被静默丢弃。 这是最坑的一条。surface 正常出现，没有任何报错，所有用 `path` 绑定的属性一律空着。我把同样的字节晚一秒重发一次，字段立刻填上。logcat 里三条消息全是 parsed OK，surface 也变成了 1 个，但值就是没落：

```text
parsed OK -> A2uiCreateSurfaceMessage surfaceId=booking
parsed OK -> A2uiUpdateComponentsMessage surfaceId=booking
parsed OK -> A2uiUpdateDataModelMessage surfaceId=booking
activeSurfaces.size=1 ids=[booking]
```

所以我最终的写法是：先发 `createSurface` 和 `updateComponents`，等 surface 出现之后再推数据模型。这刚好也是流式 agent 的真实顺序，算歪打正着。

二、catalogId 是 v0_9，不是 v0_9_1。 文档站上那段「stable」示例写的是 `version: v0.9.1` 和 `.../v0_9_1/...`。库里的真实常量是 `protocol = v0.9` 和 `.../v0_9/catalogs/basic/catalog.json`。parser 两种版本号都收，但 `createSurface.catalogId` 必须跟你注册的那个 catalog 的 id 完全一致，对不上 surface 就永远不出现，同样不报错。我的做法是干脆不从文档抄字符串，直接从库的常量里取。

三、`processInput` 不是 `A2uiMessageProcessor` 的方法。 它是 `androidx.a2ui.model.processor` 包里的顶层扩展函数，必须 `import androidx.a2ui.model.processor.processInput`，照文档那行直接抄会编译不过。

四、失败是静默的。 `processInput` 内部会把 `A2uiException` 吞掉，转成 `processor.processError(...)`，只从 `outboundEvents` 发出去。你要是没收集这个流（文档里收集了，但没说漏掉的代价），协议错误和解析失败就完全看不到，屏幕上只有一个白框。调试这个库的第一件事，是把 `outboundEvents` 接上。

五、`material3-a2ui:1.0.0-alpha01` 跟当前稳定版 Compose BOM 不兼容。 它的 POM 硬要 `androidx.compose.*` 1.13.0-alpha03，material3 还卡了个 `[1.5.0-alpha28]` 的严格区间。用稳定 BOM 解析出来的是 1.12.x，装不上。最后我把 BOM 摘掉，手动钉版本。

六、compileSdk 37 minor 1 的写法没文档。 `compileSdk = 37` 找不到已经装好的 android-37.1。能用的写法是：

```kotlin
compileSdk { version = release(37) { minorApiLevel = 1 } }
```

七、surface 的根组件 id 必须叫 `root`。 它是个私有常量，写成别的名字，你的 Column 就不是根。

八、枚举取值全靠猜。 这些值写错就是 schema 失败，而文档里没有：

```text
Text.variant           h1 h2 h3 h4 h5 caption body
Button.variant         primary secondary borderless
TextField.variant      shortText longText number obscured
```

我的办法是 `javap` 拆 AAR 里的枚举，一个个读出来。

另外还有一条跟 A2UI 无关但绕不开的：AGP 9.x 下再挂 `org.jetbrains.kotlin.android` 会被直接拒掉，官方说 AGP 9.0 起 Kotlin 支持已经内置，只需要保留 compose 插件。

## 我的判断

能跑通，而且渲染出来的确实是原生 Material 3，不是套壳网页。但 alpha 阶段的文档跟不上代码：真实的 catalogId、`processInput` 的形态、消息时序、版本钉法，全得自己拆包看。这套东西现在适合做概念验证，别急着往生产里放。

KMP 侧要再等等。只有 aar 这一件事，就决定了 Compose Multiplatform 现在只能用 Android 那一头。

我下一步想试的是把 outbound 那条路走通：用户点了 Confirm booking，事件回到 agent，agent 根据结果再发一条 `updateComponents` 改界面。这条链路我只接了线，没真按过按钮，所以这一篇里不敢说它成不成。

你会让你的 AI Chat 客户端由 agent 直接生成界面吗？还是更愿意自己写死界面、只让 agent 填内容？评论区聊聊。

参考资料：
- A2UI 协议官网：https://a2ui.org/
- Jetpack Compose 的 A2UI 渲染器（官方文档）：https://developer.android.com/develop/ui/compose/agentic
- 自定义 catalog：https://developer.android.com/develop/ui/compose/agentic/manage-catalogs
- A2UI Compose 版本页：https://developer.android.com/jetpack/androidx/releases/a2ui-compose

验证边界：两张截图、三段消息的解析日志、catalogId 与协议版本常量、消息时序的 A/B 对照、构建与安装，都在本机跑真机模拟器得到（Android 模拟器 API 36.1，APK 13,471,211 字节，构建 BUILD SUCCESSFUL）。组件属性名与枚举取值来自 `javap` 拆包。按钮点击回传 agent 那条链路没有实测；图像/视频/音频用的是占位渲染器，没有接真实加载库。
