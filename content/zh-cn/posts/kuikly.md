---
title: "同样写 Compose，腾讯这条路多养了十五万行原生代码"
date: 2026-10-30T08:10:11+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "Kotlin Multiplatform", "跨端开发"]
summary: "同样能写 Compose，腾讯选了原生渲染而不是 Skia 自绘：六端背后是四套渲染器、十五万行平台代码。"
cover:
    image: "/images/kuikly/cover.png"
    alt: "同样写 Compose，腾讯这条路多养了十五万行原生代码"
    relative: false
---

腾讯 TDS 的 KuiklyUI 今天冲上 GitHub 趋势榜，三千六百星。它是个 KMP 的 UI 框架，一码六端。

六端这个说法得拆开看。Android、iOS、鸿蒙是正式支持，Web 和微信小程序标着 Beta，macOS 是 Alpha。文档里写得清楚，别把趋势榜当成熟度。

![六端支持情况与四套渲染器](/images/kuikly/fig1.png)

它和 Compose Multiplatform 最容易混，因为两边都能写 Compose。但底下是两条完全不同的路，我把仓库扒了一遍，账能算出来。

## 一条自己画，一条租别人的

Compose Multiplatform 用 Skia 自绘。你写一个 Box，它自己拿画笔把像素涂出来，六个平台涂出来一模一样，因为画笔是同一支。

Kuikly 不画。它把 Box 翻译成各平台自己的 View，Android 上是 View，iOS 上是 UIView，鸿蒙上是 ArkUI 组件。渲染这活外包给系统。

官方那篇 Compose DSL 发布文写得很直白，原文是「CMP 采用 Skia 自绘，而 Kuikly Compose DSL 可以直接复用现有成熟的原生渲染体系，拥有与原生系统完全一致的使用体验，例如滑动手感、输入框、动画、无障碍等」。

收益和代价都在这一句里。滑动手感、输入框光标、无障碍朗读这些，系统给了，你不用自己模拟一遍，而那正是自绘框架最费劲的地方。代价是每个平台都得有一套翻译层，系统长什么样你都得跟。

![两条路的分岔：Skia 自绘与原生渲染的对照](/images/kuikly/fig4.png)

## 账算出来是这样

我把仓库 clone 下来，按目录数了一遍行数。

共享的 Kotlin 二十万九千行，其中 core 三万三千行，剩下十七万五千行在 compose 目录。

各平台的渲染器加起来十五万五千行。Android 三万一千行 Kotlin，iOS 三万七千行 Objective-C，鸿蒙五万一千行 ArkTS 加 C++，Web 三万五千行 Kotlin。

一份共享代码，配四套平台渲染器。这个比例是这条路线的固定成本，不是腾讯写得糙。CMP 那边这笔账为零，因为它只有一支画笔要维护。

鸿蒙那五万一千行值得单看一眼，其中 ArkTS 只有五千九百行，剩下的四万两千行是 C++ 和 C。ArkUI 的调用要走原生层，光靠 ArkTS 接不住这个量。这也说明原生渲染这条路，越往深走越靠近系统那一端。

![共享代码与平台渲染器的行数对比](/images/kuikly/fig2.png)

## 那十七万五千行不是它写的

compose 目录是从 JetBrains 的 compose-multiplatform-core 分叉出来的，基于 Compose 1.7.3。

README 里自己交代了，把包名从 androidx.compose 改成 com.tencent.kuikly.compose，用不上的功能注释掉，不一路升级到底，方便以后跟上游。

所以它跑 Compose 不是兼容，是把 Compose 搬过来再改。官方敢说「兼容百分之九十五以上的 Jetpack Compose API」，因为物理上就是同一份代码，改的是对接层。

## 它先有自己的 DSL，Compose 是后接的

这点很容易被忽略。Kuikly 一开始根本没有 Compose，它有一套自研 DSL，用 ViewBuilder 搭界面，页面用 @Page 注解标出来，状态靠 observable 装饰器。

demo 目录里两种写法并存，自研 DSL 的页面有三百个，Compose 写的一百二十三个。数量对比说明主次，Compose 是后来接上去的一层，不是地基。

我翻了一个自研 DSL 的页面，写法是这样，跟 Compose 完全两个路子。

```kotlin
@Page("NetworkExamplePage")
internal class NetworkExamplePage : BasePager() {
    private var output by observable("")
    override fun body(): ViewBuilder = { NavBar { attr { title = "网络请求示例" } } }
}
```

同一件事，Compose 那边写的是 `@Composable` 加重组，自研这边是声明式的一棵树。两套并存意味着这个框架要维护两条前端的口子。

## 真正的活干在中间那层

Compose 重组时不停增删改节点，这些改动要通过 Applier 交给渲染后端。Android 和 Skia 都有官方实现，Kuikly 得自己写一个。

它的 KuiklyApplier 继承 AbstractApplier，每个 KNode 一边挂 LayoutNode 管布局，一边挂 DeclarativeBaseView 管渲染，一对一。

麻烦在坐标系，官方举的例子一行代码就露馅。

```
Box(Modifier.padding(10.dp).size(50.dp).background(Color.Blue))
```

background 绑在里层坐标系，画出来是 50×50。加上 padding 之后外层边界已经是 70×70。而 Kuikly 的 View 只有一个 frame，你把哪个数填进去都不对。

填外层，背景跟着铺满 70×70，画大了。填里层，尺寸对了，padding 偏移丢了，子节点全错位。给每个 Modifier 建一个 View 更不行，三层 Modifier 就三层嵌套，页面一复杂层级就炸。

![一个 View 只有一个 frame](/images/kuikly/fig3.png)

定下来的规矩是「绘制坐标系决定 Size，相对位置决定 Position」。一句话背后是两套渲染语义的对齐，这类框架真正的时间都花在这儿。

## 十一个构建脚本

repo 根目录摆着十一份 Gradle 脚本，从 Kotlin 1.3.10 一直排到 2.1.21，每份都配一个同名 settings 文件。

不是因为喜欢整齐。Kotlin Multiplatform 的 Gradle 插件 API 这些年一直在变，一套脚本吃不下所有版本，只能每个大版本备一份，用的时候切过去。

这个细节比任何性能宣传都有说服力。国内大厂做长期框架，最大的成本是三年后还有人在用老版本 Kotlin，你得让他也能跑起来。

## 许可证里有一条要看清

LICENSE 主体是 MIT 那套，但多了一句限制。原文的关键词是 `may not modify, extend, or create derivative works`，目的限定在 `adding or enabling dynamic code loading / live updating capability`，而且要先拿到腾讯的书面同意。

翻过来就是，动态下发和热更新这套能力，没有腾讯书面同意，你不能自己改出来。

而动态下发恰恰是它最大的卖点。Compose Multiplatform 编译期出包，改一行要重新发版；Kuikly 能把页面当资源下发，改完直接生效。这个差距对国内业务的吸引力，比任何性能数字都大。

GitHub 上这仓库的 license 字段显示 NOASSERTION，原因就在这，它不是一份标准协议。

## 谁适合用

官方给的验证数据是内部三十多个 App、五亿日活场景，外部二十多个团队接了，包括快手、方正证券、MiniMax。迁移成本他们写着五十多个标准 Compose 页面一次性搬完，还有用 AI 生成了七十三个功能 demo，零修改编译运行。

要鸿蒙或者微信小程序，这是现在最省事的一条路，CMP 还没覆盖。只做 Android、iOS 和桌面，自己养四套渲染器这件事不太划算，那十五万行是要你替它维护的。

我这边只验证到工程结构和代码量这一层，外加编译器边界。gradle 配置能过，十三个模块，两分四十九秒；core 模块编译成 macOS arm64 的原生库也跑通了，一分零六秒，JDK17 加 Kotlin/Native 就够，不用 Android SDK。

六个端的完整 App 没跑，那要 Xcode 配 CocoaPods 和 DevEco Studio 各自一套。

你要是选跨端框架，会先问它覆盖几个端，还是先问它改一版要发几次包？
