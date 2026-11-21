---
title: "Compose Hot Reload：写崩了，不用重启"
date: 2026-11-21T15:00:59+08:00
draft: false
categories: ["公众号"]
tags: ['Kotlin', 'Compose Multiplatform', 'JetBrains']
summary: "热重载撞上运行时异常会怎样，以前没人说清。我在最小工程上故意写坏三次：能编译但运行时炸、编译不过、再修回来，三种收场各不相同。端到端 768 毫秒里有八成花在编译上。"
cover:
    image: "/images/cmp-hotreload/cover.png"
    alt: "Compose Hot Reload：写崩了，不用重启"
    relative: false
---

我在自己搭的一个桌面小窗口里敲了一行 `error("故意写坏")`，保存，然后盯着屏幕等它炸。

它炸了。窗口里的字没了，换成一片红，控制台吐出一句 `Error was captured in composition while live edit was enabled.`，紧接着是我写的那句中文异常。应用进程没退，窗口也没关。

这件事在一年前是不可能的。热重载撞上运行时异常，轻则卡在半个旧界面上，重则整个进程一起走掉，你只能重跑一遍应用，把状态从头点回来。Compose Hot Reload 1.3.0 把这套收场改掉了，它 11 月 20 日上了 Maven Central，是这个项目的第一个稳定版。

## 这一版 Compose Multiplatform 只改了一件事

先说清版本关系，不然容易对不上号。

Compose Multiplatform 1.13.0-rc01 在 11 月 18 日打 tag，桌面侧的全部特性就一条：把 Compose Hot Reload 提到 1.3.0-rc01。其余都是修问题，字体缓存、内存泄漏、`measureAndLayout` 崩掉这类。两天后热重载自己发正式版。

所以这一轮值得看的就是热重载本身。它的 1.3.0 发版说明开门见山，主题写着 error recovery：改坏了之后，它要能认出坏在哪、说清坏成什么样，然后在你把源码修好的那一刻自己回来。

![Compose Hot Reload 1.3.0 的三条改动：错误上报与恢复、hard reset、给 agent 的那一套](/images/cmp-hotreload/fig4.png)

## 三种坏法，三种收场

我在同一个工程上把三种坏法各试了一遍，它们的下场并不一样。

| 坏法 | 编译 | 界面 | 你要做什么 |
|---|---|---|---|
| 能编译，运行时抛异常 | 通过，514ms | 错误浮层接管，进程活着 | 改回源码再保存 |
| 编译不过 | 失败，468ms | 保持上一版，不受影响 | 改到能编译 |
| 修回来 | 通过 | 新版本接管 | 什么都不用做 |

第二种最容易被误会成没问题：编译失败时热重载干脆不应用，你看到的是上一次成功的界面，而不是半坏状态。它拒绝的方式也很干脆，`gradle reload` 直接 BUILD FAILED，应用继续跑，一个字节都不换。

第一种最要紧。以前这种错误要么被吞掉、界面停在崩溃前那一帧，要么把整个 composition 拖死；现在错误被 composition 层接住并上报，浮层里给出异常类型和堆栈，窗口还在。

![应用 stdout 里的两行：错误被 composition 捕获，下面那段是编译失败时的 gradle 输出](/images/cmp-hotreload/fig3.png)

## 改完到看见新界面，我量了几次

计时口径是端到端：我用脚本改文件、记录写入时刻，应用里一个 `LaunchedEffect` 在每次新版本真的被 composition 用上时往文件里打一行时间戳，两边相减就是「改完到看见」的墙钟耗时。每轮都真的触发增量编译，不是缓存命中。

![三种改动各自的端到端耗时：只改文本 768 毫秒、改结构 721 毫秒、报错后修好 745 毫秒](/images/cmp-hotreload/fig1.png)

| 场景 | 次数 | 中位 |
|---|---|---|
| 只改文本（叶子节点） | 7 | 768ms |
| 改结构（插一个 Button） | 5 | 721ms |
| 报错之后修好 | 2 | 745ms |

三次中位数都在 750 毫秒上下，和我之前在别处看到的「热重载 200 毫秒」不是一个量级。差的这部分不在热重载本身，在编译：`gradle reload` 自报的构建时间落在 528 到 726 毫秒，一个只改了字符串常量的 Kotlin 文件也要重走一遍编译和类加载。热重载省掉的是重启应用和重建状态，编译这一截它替你省不了。

那 200 毫秒指的是应用侧的 class redefinition 加 recomposition，不包括你按下保存之前的编译。写文章时把这个口径说清楚，比报一个更好看的数有用。

![七次只改文本的端到端拆分：编译占八成，其余一百多毫秒是重载应用与重组](/images/cmp-hotreload/fig5.png)

## 它是怎么把界面救回来的

发版说明给的是机制名：Recomposer 的 error-state API，Compose Runtime 1.11.0 起就有，但错误能被恢复要等 Compose Multiplatform 1.13.0。两边版本一起对上，subcomposition 里抛的错才算「可捕获且可恢复」。

另一条是新加的 hard reset：状态被一次错误的重载弄坏之后，它干脆把应用窗口重建一遍，而不是在坏状态上继续打补丁。这个策略现在是 Reset UI 的默认行为，想退回轻量的那种重置，加一个 `-Dcompose.reload.compositionHardResetEnabled=false` 就够。

窗口重建听着粗暴，但它解决的是一个真问题：热重载最怕的不是报错，是错完之后状态不一致，界面看着正常、点哪儿都不对。

## 顺带补上的几处

1.3.0 还改了三样跟 agent 有关的东西，值得记一笔。MCP 的 `take_screenshot` 换成了 Compose Desktop 自己的 `captureContentToImage()`，不再借 AWT 的 Robot 抓屏幕，在 Compose Multiplatform 1.13.0 上更准。

语义树里可滚动的节点补了 `scrollX`/`scrollY`，agent 终于能知道列表滚到哪儿了。外加一组实验性的 headless 工具，起一个无窗口的会话跑界面，opt-in 才开。

## 门槛在前面，不在功能上

想自己跑一遍，会先撞一个跟功能无关的墙：它要 JetBrains Runtime 21。我第一次裸跑，报错写着 `Failed to find suitable JetBrains Runtime 21 installation on your system`。

我机器上唯一的 JBR 是 Android Studio 自带的 25.0.3，版本对不上。打开自动下载（`-Pcompose.reload.jbr.autoProvisioningEnabled=true`）之后它自己把 JBR 21.0.10 拉了下来。

门槛过了之后，剩下的体验就一句话：改完、保存、八百毫秒之内看见。

我的判断是，热重载这条线的价值正在从「快」挪到「摔不坏」。快到 200 毫秒的收益是有上限的，人眼分辨不出 200 和 400；而一次崩溃之后能不能原地站起来，决定的是你敢不敢在真实工程里开着它改代码。

顺着这个方向想，如果哪天 agent 也能自己改代码、自己触发重载、自己判断界面有没有坏，那套 headless 工具和语义树就比速度更要紧。

你平时用热重载是随手开着，还是只在小 demo 里试试？敢在主工程开着的，回一句，我把你们撞过的坑整理成下一篇。
