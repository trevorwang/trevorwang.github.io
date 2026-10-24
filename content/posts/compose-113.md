---
title: "Compose 1.13 发布了，但真正值得写的是 1.12"
date: 2026-10-24T16:26:16+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "Compose Multiplatform"]
summary: "1.13 这版只有一个新功能，其余全是修问题。真正重的东西藏在 1.12 里，热重载接上了 MCP。"
cover:
    image: "/images/compose-113/cover.png"
    alt: "Compose 1.13 发布了，但真正值得写的是 1.12"
    relative: false
---

10 月 23 日，Compose Multiplatform 1.13.0-beta01 发了。我本来打算直接做升级实测，翻完 changelog 又改了主意。

这一版改的东西不少，但没一个是新方向。真正重的东西，藏在两个月前的 1.12 里。

先说 1.13 改了什么。

## 这版只有一处算新功能

桌面端 ComposePanel 在 SwingGraphics 渲染模式下，现在可以跟着显示器的刷新率走帧了。默认关，打开要设一个系统属性。

```kotlin
System.setProperty("skiko.swing.frame.pacing", "true")
```

Linux 上可能还要加 forceTimer，用系统定时器补 vsync 信号，精度差一点但能用。

Swing 里嵌 Compose 的画面，以前是带自己节奏的，跟屏幕刷新对不上。这次算补上了。

iOS 那边加了 UIKitInteropSafeAreaInsetsPolicy，可以控制嵌进来的 UIKit 组件能感知多少安全区。混编 App 一直缺这个开关。

至于多端通用的改动，只有一条。AnnotatedString 现在能渲染 Bullet 注解了。

剩下的全是修问题。

iOS 修了键盘打开时侧滑返回会崩、UIStepper 在重叠安全区的 interop 视图里位置不对、弹层渲染残留、Mac 上跑 iOS App 时从右往左选中文本会崩。四条里两条标着 prerelease fix，说明是 alpha02 自己引出来的。

Web 那边集中在 Safari。iOS 27 上输入框选不中、焦点丢了恢复不回来、多行输入框第一次点光标跳末尾。还补了个字体 fallback URL 的自定义配置。

桌面端只修了一个，Windows 上开了 D3D 同步拖拽时窗口缩放卡顿。

顺带说下这次版本号的乱。

Material3 从 1.12.0-alpha03 跳到 1.5.0-beta01，看着像倒退，其实是 JetBrains 把 Material3 的版本号对齐到 Jetpack 那边去了。官方对照表里 Material3 那行，写的就是 Jetpack 的 1.5.0-beta01。

Navigation 从 2.10.0-rc01 变 2.10.0，Navigation3 从 1.2.0-rc01 变 1.2.0，两个 rc 都收了尾。Material3 Adaptive 从 1.3.0 到 1.4.0-alpha01。

所以这次升级真正要留意的，不是 API 动了什么，是你 gradle 里那几个版本号要不要跟着动。

这一条没人写在升级说明里。

## 有一处得注意

升级说明里只有一条硬提醒。

桌面端用了 experimental 的 DragAndDropTransferAction 的库，必须重新编译，因为二进制不兼容。

注意是库，不是你的应用。你要是自己不写拖拽，问题一般出在依赖里某个第三方库上。

## 我扫了自己的三个工程

实测编译没跑通。1.13.0-beta01 的插件在 Maven Central 上能直接 curl 到，Gradle 在这台机器上就是解析不到，换了阿里云镜像和 JetBrains 的 dev 仓库都没解决，折腾几十分钟没结果。

但代码是能扫的，而且这比编译结果更早告诉你有没有风险。

三个 Kotlin Multiplatform 工程，加起来 492 个 kt 文件。

NativeCanvas 和 NativePaint，也就是 1.12 里把废弃级别提到 ERROR 的那两个类型，零处。

SwingPanel 和 ComposePanel，零处。

拖拽相关的，只有两处，在 Piko 那个项目的 MagnetDropTarget.kt 里。用的是标准的 dragAndDropTarget，配 AWT 的 Transferable 判断数据类型。不是这次改的那个 API。

所以从 API 面看，这三个工程升到 1.13 没有卡点。真要升，唯一要动的是仓库配置，beta 版得把 dev 仓库加回来。

这个结论对写文章不算精彩，但它恰好说明一件事。这类框架的版本升级，风险很少在你想的地方。

## 翻回 1.12，才看见重的东西

1.12.0 是 8 月 25 日发的，Highlight 只有一行。

Compose Hot Reload 加了个 MCP server，AI agent 可以连到正在运行的 Compose 应用上实时操作。

这句话当时没几个人讨论。它的实际分量，得看工具清单。

一个 Gradle 任务 hotMcpServer 把服务拉起来，连上应用、等启动、检测关闭、断了自己重连。接进来的 agent 有这些工具：

status 看应用和重载状态，包括上一次报错的内容。reload 和 await_reload 触发热重载并等它完成。restart 和 reset_ui 重启应用或重置界面。

take_screenshot 截图，可以指定保存路径。list_windows 和 resize_window 看窗口、改窗口大小。get_semantic_tree 读语义树，弹窗、底部弹层、popup 都算在里面。get_ui_error 和 get_logs 读当前错误和应用日志。

还有 click、type_text、scroll 这一组，直接模拟用户操作。

接起来不复杂。工程里跑 gradlew hotMcpServer 把服务拉起来，把这个端点告诉你的 agent 客户端，剩下的它自己连。

官方给的场景是让 agent 改一个 Composable，重载，读语义树确认按钮还在原位。中间没有人参与。

![Compose Hot Reload 暴露给 agent 的工具清单，左边是 1.12 引入的，右边是 1.13 的改动](/images/compose-113/cmp113-tools.png)

## 这条链路补上的是验证

把上面几件事串起来，一个 agent 能做什么，就很清楚了。

改代码，触发重载，截图看结果，读语义树确认结构对不对，读日志确认没报错。不对就再改。

整个过程人不用进去。

一个 agent 自己把这一圈走完，这就是闭环。

上一版 1.3.0-beta02 又把 scrollable 节点的信息补全了。能滚的节点现在带 scrollX 或 scrollY，给出当前偏移、最大偏移，还有内容反向滚动时的 reversed 标记。

这条看着琐碎，但对 agent 很关键。以前它只知道这里有个可滚动的容器，现在知道滚到哪了、还能不能往下滚。

## 和 Flutter 那篇说的是同一件事

上个月 Flutter 团队那篇官方文章，核心判断是代码生成早就不是瓶颈了，验证才是。

他们给的解法是天生的。热重载 200 毫秒，widget test 500 毫秒，状态检查直接读 Widget 树。

JetBrains 走的是另一条路。它不在框架层面追求快，而是把运行中的应用开放给 agent，让 agent 自己动手验。

一个靠框架本来就轻，一个靠把门打开。

我倾向于认为后者更通用一点，因为它跟框架无关，只要求运行时能被程序和 agent 访问。

但这也意味着，这条路线的收益上限，取决于那个 MCP server 能暴露多少东西。现在这十几个工具够用，往后要处理更复杂的界面，还不够。

所以我的问题给到你们：你手上的 Compose 工程，试过让 agent 直接接管热重载和验证吗？还是仍然是自己改完自己点？

再问一句，你觉得这种把运行时开放给 agent 的路子，Web 端和移动端会不会比桌面走得更慢？桌面有 JVM 可以直接接，移动那边得另想办法。
