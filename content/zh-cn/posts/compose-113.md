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

10 月 23 日，Compose Multiplatform 1.13.0-beta01 发了。

标题里这句话不是标题党。这一版改的东西不少，但没一个是新方向，真正能把跨端这件事往前推一步的东西，两个月前就已经悄悄上过了。

先说结论，1.13 只有一处算新功能，剩下全是修问题。一句话带过就够了。

## 1.13 就这点东西

桌面端 ComposePanel 在 SwingGraphics 渲染模式下，现在能跟着显示器刷新率走帧。默认关，要开就设这个属性。

```kotlin
System.setProperty("skiko.swing.frame.pacing", "true")
```

iOS 加了 UIKitInteropSafeAreaInsetsPolicy，能控制嵌进来的 UIKit 组件感知多少安全区。

多端通用只有一条，AnnotatedString 能渲染 Bullet 注解了。

其余全是修 bug。iOS 四个，Web 集中在 Safari 的选中和焦点，桌面端只修了一个 Windows 的缩放卡顿。

好，1.13 的部分到此结束。

## 真正该看的是两个月前那行字

8 月 25 日，1.12.0 发布。Highlight 只有一行。

Compose Hot Reload 加了个 MCP server，AI agent 可以连到正在运行的 Compose 应用上实时操作。

这行字当时几乎没引起讨论。但把它的工具清单摊开看，就知道这不是个小更新。

## 一个 agent 拿到手的到底是什么

工程里跑一个 Gradle 任务 hotMcpServer，服务起来，连上应用，等启动，检测关闭，断了自己重连。

接进来的 agent，能调这些工具。

状态和重载这一组，status 看应用和重载状态，连带上一次报错的内容一起给。reload 和 await_reload 触发热重载并等它完成。restart 和 reset_ui 重启应用或者重置界面。

观察这一组，take_screenshot 截图，能指定保存路径。list_windows 和 resize_window 看窗口、改窗口大小。get_semantic_tree 读语义树，弹窗、底部弹层、popup 都算在里面。get_ui_error 和 get_logs 读当前错误和应用日志。

操作这一组，click、type_text、scroll，直接模拟用户输入。

![Compose Hot Reload 暴露给 agent 的工具清单](/images/compose-113/cmp113-tools.png)

这三组拼起来，第一次让 agent 在 Compose 里有了完整的闭环。

## 闭环是怎么闭上的

以前 agent 写 Compose 代码，写到界面上就断了。它能改文件，能编译，但看不到界面长什么样，点不了按钮，也不知道这次改动到底对不对。

人得接手。这就是那一大段等待。

现在这个循环是这样跑的。agent 改代码，调 reload 触发热重载，take_screenshot 看一眼，get_semantic_tree 读一遍结构确认布局没塌，get_logs 确认没抛异常。不对就再改一轮。

整个过程人不用进去。

一个 agent 自己把这一圈走完，这就是闭环。

1.3.0-beta02 又补上了缺的那块。可滚动的节点现在带 scrollX 或 scrollY，给出当前偏移、最大偏移，还有内容反向滚动时的 reversed 标记。

这条看着像管道工活。但对 agent 很关键。以前它只知道这里有个可滚动容器，现在知道滚到哪了、还能不能继续往下滚。不然它截图截到的永远是第一屏。

## 这跟 Flutter 是两条路

上个月 Flutter 团队那篇官方文章，核心判断是代码生成早就不是瓶颈，验证才是。

他们的解法是框架天生就轻。热重载 200 毫秒，widget test 500 毫秒，状态检查直接读 Widget 树，全是确定的、结构化的数据。

JetBrains 没走这条路。它不去优化框架本身的速度，而是把运行中的应用整个打开给 agent，让 agent 自己动手验。

一个靠轻，一个靠开门。

我倾向后者更通用。它跟框架无关，只要求运行时能被程序访问。但这也意味着天花板很明确，取决于那个 MCP server 能暴露多少东西。现在十几个工具够用，往后要处理的界面更复杂，肯定不够。

## 所以现在能做什么

如果你手上有 Compose 桌面工程，今晚就能试。

前提是 Compose Hot Reload 1.2.0 以上。工程里跑 gradlew hotMcpServer，把这个端点接到你的 agent 客户端里。让它改一个 Composable，重载，读语义树确认按钮还在原位。

上面这些工具清单和闭环路径，是 1.12 和 Hot Reload 1.3.0-beta02 的 release notes 加上仓库源码读出来的，不是我在自己机器上跑过一遍。

这点说清楚，免得你以为是实测。

按我的经验，这条路第一次接最容易卡在两个地方。MCP server 只在 dev 模式下起，你的 client 得连对端口。还有一个坑，get_semantic_tree 在弹层上的表现，官方文档写得很简，实际用之前先单独试一次。

第一次看到它自己把这一圈走完，感受跟看它生成代码完全不同。

问题给到你们。你手上的工程，试过让 agent 直接接管热重载和验证吗？还是仍然自己改完自己点？

再问一句，你觉得这种把运行时开放给 agent 的路子，Web 端和移动端会不会比桌面走得慢？桌面有 JVM 能直接接，移动那边得另想办法。
