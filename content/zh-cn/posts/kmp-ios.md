---
title: "JetBrains 撤掉了 KMP 插件里的 Swift 支持：编译链一条没动"
date: 2026-10-06T10:44:56+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin"]
summary: "10 月 5 日 JetBrains 发了篇不长的公告，标题直白：KMP 插件里停掉 Swift 语言的 IDE 支持。生效版本写得很具体，IntelliJ IDEA 2026.3 和 Android Studio Rabbit 2 | 2026.2.2。"
cover:
    image: "/images/kmp-ios/cover.png"
    alt: "JetBrains 撤掉了 KMP 插件里的 Swift 支持：编译链一条没动"
    relative: false
---

## 先说这条公告的字面意思

10 月 5 日 JetBrains 发了篇不长的公告，标题直白：KMP 插件里停掉 Swift 语言的 IDE 支持。生效版本写得很具体，IntelliJ IDEA 2026.3 和 Android Studio Rabbit 2 | 2026.2.2。

撤掉的功能清单也具体，我抄一下：

```text
This includes editing, syntax highlighting, and navigation within Swift
files and between Swift and Kotlin.
```

> 译文：包括 Swift 文件里的编辑、语法高亮，以及 Swift 文件之间、Swift 与 Kotlin 之间的跳转。

看到这几行，很多人的第一反应是 KMP 的 iOS 是不是要凉。先别急，同一篇公告的第二段就把边界划出来了。

## 撤掉的是哪一层

原文第二段是这么写的：

```text
Running iOS applications and debugging Kotlin code on iOS will remain fully
supported. Kotlin/Native functionality, such as Swift Export and importing
Swift packages into Kotlin projects, is not affected.
```

> 译文：运行 iOS 应用、在 iOS 上调试 Kotlin 代码，仍然完整支持。Kotlin/Native 的功能，比如 Swift Export 和把 Swift 包导入 Kotlin 项目，不受影响。

也就是说，被拿掉的是 IDE 里读改 Swift 的那一层。编译器、Native 工具链、跟 Xcode 对接的那条路都还在。

后半句我在这台机器上验了一遍。本机已经有 Kotlin/Native 2.4.0 的预编译包和 Xcode 26.2，我拿它把一份小代码编成 iOS 的 framework：

```bash
kotlinc-native -target ios_arm64 -produce framework -o out/shared src/shared.kt
```

九点八秒，产物是一个 Mach-O 64-bit arm64 的动态库，外加一份给 Xcode 用的头文件。这条链一步没退，撤 Swift 功能没动它。

![](/images/kmp-ios/ios-framework.png)

## 那 iOS 到底靠哪几条链跑起来

把链路拆开看，顺序是这样：

1. Kotlin/Native 把共享代码编成 Apple 能用的 framework，就是我上面那九点八秒。
2. Xcode 把这个 framework 链进 iOS 应用，Gradle 里那条 `embedAndSignAppleFrameworkForXcode` 干的就是这件事。
3. Swift 壳调 Kotlin。现在主要走 Objective-C 头文件那套互通，官方另外在推 Swift Export，还在 Alpha，目标是让 Kotlin 直接导出成 Swift 模块，省掉头文件那一层。
4. IDE 里读改 Swift 只是开发时的顺手，撤掉它，上面三层一步没动。

想明白这四条，公告的轻重就出来了。

## 他们为什么撤

官方给的理由是使用量。他们的埋点和用户调研显示，做 KMP 项目的 iOS 开发者多数用 Xcode 写 Swift，而在 KMP 插件里干活的人越来越多用 Kotlin 和 Compose Multiplatform。

还有一层历史包袱：这套 Swift 工具最初是给 AppCode 开发的，而 AppCode 早就停了；后来挪到 Fleet，再挪进 KMP 插件。要让它的核心架构跟上 Swift、Xcode 和 IntelliJ 平台的持续变化，投入不小。他们权衡之后决定不做，并且写明目前不打算用别的功能替代。

这个取舍其实跟 KMP 这几个月的动作是一条线：他们把力气花在让 Kotlin 侧更好，而不是替 Xcode 干活。

## 社区那边的进度，比二手报道新

同一天热榜上还有另一条，方向正好相反。开源 OSM 编辑器 StreetComplete 的 iOS 版进了 TestFlight，这是它多年移植工作后的第一个可测版本。

这个移植从 2023 年 12 月开的追踪 issue 算起，到现在还 open 着，累计 51 条评论，维护者说全量移植大概是一人年的工作量，2024 年上半年靠 Prototype Fund 的资金全职推进，到当年 9 月完成约一半。

二手报道里有个说法已经过期了。它写 Compose Multiplatform 的 MapLibre 地图库还不存在，是移植的卡点。我去读了 StreetComplete 的构建文件，现在写的是这行：

```kotlin
implementation("org.maplibre.compose:maplibre-compose:0.19.0")
```

这个库在 Maven Central 上从 0.11.0 一路发到了 0.19.0，那个缺口已经补上。

代价也得说清楚。原型基金到期了，维护者在 issue 里写后续资金不乐观，OpenStreetMap 没钱，NLNet 的项目也在收缩，众筹这几家赞助没涨。已知问题还挂着：iOS 15.x 上地图不工作，表单只能用手势返回，bug 还多。

## 我数了一遍它的仓库

顺手把这个仓库按源集数了一遍，比看新闻有说服力：

- commonMain：1,281 个 Kotlin 文件，3.6MB
- commonTest：287 个文件，1.3MB
- androidMain：34 个文件，43KB
- iosMain：33 个文件，42KB

也就是说，两端各补四十多 KB 的壳，就把同一个 app 接到了 iOS 和 Android 上。iOS 侧那 33 个文件里，最大的一块是 IosModule.kt，六千字节，干的是依赖注入装配，剩下的都是网络、同步、上传、周期清理这类平台专属件。

![](/images/kmp-ios/source-sets.png)

看清这个比例，公告的影响面也就清楚了：撤掉的功能，作用范围就是这四十二 KB 的壳，和共享的三点六 MB 无关。

## 这两件事其实是一件事

JetBrains 的选择是「Kotlin 侧我做好，Swift 侧交给 Xcode」。StreetComplete 的做法正好配合这个分工：共享逻辑和 UI 都写在 Kotlin 和 Compose 里，Swift 那边只留一层壳。

所以判断很简单，问自己几句：

- 你项目里的 Swift 平时在哪儿写。本来就是 Xcode 开着，这条公告对你几乎没影响。
- 你要是在 Android Studio 里统一读改 Swift，那从现在起要改流程，或者用旧版本。
- 别把 IDE 支持和编译链支持混着看。前者撤了，后者没动。
- 技术路线通，不等于项目能到终点。StreetComplete 这一条提醒的是钱和人。

## 手上这个版本还能用多久

门槛是 Android Studio Rabbit 2 | 2026.2.2 和 IntelliJ IDEA 2026.3，本机装的是 2026.2，所以你现在手里那些 Swift 功能还在，只是往后不再更新。急着用就晚点再升，不急就随它去，反正 Swift 真正的家一直是 Xcode。

你的 KMP 项目里，iOS 那侧的 Swift 是在哪儿写的？评论区聊聊。

参考资料：
- https://blog.jetbrains.com/kotlin/2026/10/discontinuing-swift-language-ide-support-in-the-kotlin-multiplatform-plugin/
- https://github.com/streetcomplete/StreetComplete/issues/5421
