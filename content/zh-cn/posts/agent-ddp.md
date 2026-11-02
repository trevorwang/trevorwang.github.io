---
title: "给 AI 装上 DevTools：它能看见每一帧"
date: 2026-11-01T11:13:00+08:00
draft: false
categories: ["公众号"]
tags: ["Android", "AI Agent", "DevTools"]
summary: "同一块屏幕，截图 77 KB 让模型去猜，ddp tree 只用 4,184 字节就把结构说清楚。我在模拟器上给它做了对照实测，也撞上了它现在还做不到的地方。"
cover:
    image: "/images/agent-ddp/cover.png"
    alt: "给 AI 装上 DevTools：它能看见每一帧"
    relative: false
---

AI 写前端早就不是猜的。Chrome DevTools Protocol 摆在那儿，agent 能读 DOM、能点按钮、能量颜色，改完自己回头验收。

写原生 App 就不是这回事。iOS 和 Android 没有等价的协议，agent 拿到手的只有一张截图。

截图能看，但看不出「这个按钮的边框从 5.25 变到 15.75 用了 1.8 秒」，也看不出「这行字被一个半透明层盖住了」。它只能猜，猜完就改，改完再截一张。

十月二十三号，有个叫 agent-ddp 的项目开了源，想补的就是这块。

## 它把界面变成几百行文本

装完之后，agent 问一句屏幕上有啥，拿到的是这个：

```
@e1 animatedButton "Selected" [383, 158, 315, 137]
  z=1 layer fill #0000FFFF corner 31.5
  z=2 bitmap [467, 195, 147, 63]
  z=3 layer border 15.75 #FFA500FF corner 31.5
@e2 flashBox "Flash" [460, 358, 160, 115]
```

每个元素都有引用号 `@e1`，下一条命令可以直接用。填充色、边框、圆角、坐标、层级，全是数字。

我量了一下：这个界面 25 个元素、59 个绘制节点，`ddp tree` 输出 4,184 字节。同一块屏幕截图存下来是 77,397 字节。

差 18.5 倍，而且差的不是大小，是确定性。截图要模型自己看，文本它直接读。

![同一块屏幕：77 KB 的截图和 4,184 字节的文本](/images/agent-ddp/fig1.png)

## 本机跑起来：从装到出结果

我用的是 Mac 上现成的模拟器，Medium Phone，API 36.1。

`npm i -g agent-ddp` 装 CLI，一个 Rust 编译的原生二进制，没有运行时依赖。样例工程用 Gradle 编出 10 MB 的 debug APK，装进模拟器，打开。

第一步跑 `ddp doctor`，它把设备、应用、库、端点全查一遍：

> ok   agent-ddp library in the build
> ok   control endpoint answers
> ok   25 elements on screen
> ok   hook compose.zSortedChildren (internal): answered 4608 times

二十来行全绿。这一步很关键，因为它不只查「连没连上」，还列出每个隐藏 hook 被调了多少次。后面会说为什么。

接着我只敲了一行 `ddp check all`，2.7 秒，扫了 116 帧，报出九个问题，退出码 1。

## 它报的九个问题长这样

- 两处文字被盖住：`boxCovered` 的字在它的框里一个像素都找不到
- 一处圆角越界：`hatchCard` 的灰线画到了圆角外面
- 一处动画不停：`clockBar` 最后 500 毫秒还在变，29 次
- 两处无障碍没标签、两处点击区只有 37×47，低于 48×48
- 一处对比度 1.5:1，低于 4.5:1

每一条都带证据坐标，比如那句「它的字在 [42, 1283, 258, 85] 里一个像素都不出现，那个位置屏幕是 #9E9E9EFF」。

我核对了截图，那块地方确实是灰色方块压在文字上。

![一条 check all 报出的九条问题，退出码 1](/images/agent-ddp/fig2.png)

## 我最喜欢的是它拒绝我的那一下

界面上那个被盖住的按钮，我试着让它点：

> ddp: boxCovered is covered by pointerInput: no point of it takes a touch
> next: ddp check tappable, or ddp tap boxCovered --force to touch it anyway

它不点，并且告诉你为什么，还给出两个下一步。

凡是做过 UI 自动化的人都知道这有多反常。传统框架碰到这种情况会硬点下去，然后你去猜为什么没反应。

## 它给自己定的三条硬规矩

翻文档时我发现，这个项目把三句话写成了不可违反的规矩，排在所有其他规则前面。

第一条，零误报。一条报警只有在确定有问题时才报，拿不准就什么都不说。理由写在文档里：一次误报会教 agent 养成随手加 `--force` 的习惯，那比漏报贵得多。

第二条，平台引擎是唯一事实源。什么吃到点击、画了什么、谁盖住谁，这些答案必须来自 UIKit、Compose、View 分发这些平台自己的代码。它不许自己重写一套规则去模拟平台的判断。

第三条，每个 hook 都必须计量。凡是用了私有符号或者框架内部成员的地方，都要登记，并且每次调用都计数。hook 失效时，结论变成「未知」，不许变成「有问题」。

`ddp doctor` 里那行 `answered 4608 times` 就是这条规矩的产物。系统升级后某个内部结构变了，计数会直接掉下去，你立刻知道有个检查悄悄失明了。

![写在所有规则前面的三条硬规矩](/images/agent-ddp/fig4.png)

## 它写了 CDP 方案，当天就自己否了

这个项目有份决策记录特别值得看。

最早的设计是让 Debug 包里的库直接说 Chrome DevTools Protocol，这样 Chrome 和现成的 CDP 客户端都能连上，agent 也不用学第二套协议。

同一天，它自己把这份方案否了。

理由很实在：为了模仿浏览器，它得在原生元素树上面假装出一个 `#document > html > body`，得支持 CSS 选择器，得给偏好设置起名 localStorage，还得为 Chrome 要的字段填假值。

每一样都是翻译层，学会它、维护它的人是你。更糟的是，这些翻译会把 App 真实的结构藏起来。

换掉之后，协议名改成 `screen.elements`、`input.touch`、`prefs.write` 这种原生叫法。删掉的浏览器层大约六百行。

## 装进去的东西很轻

iOS 那份是 Swift，四千三百多行；Android 那份 Kotlin，四千一百多行；CLI 是 Rust，九千七百多行。

Debug 包里的库自己就是服务器，跑在一个本地 WebSocket 上。Android 用的是只有 shell 用户和 root 能连的抽象 socket，iOS 只监听 127.0.0.1，还挡掉非 localhost 的 Host 头。

没有守护进程。每条 `ddp` 命令都是一个短进程，录下来就是一串 jsonl 文件。同一个文件，可以反复问不同的问题。

Release 包里这些代码一行都没有，Android 是 `debugImplementation`，iOS 整个包在 `#if DEBUG` 里面。

![Debug 库、Rust CLI、agent 三段，中间只有一个本地 WebSocket](/images/agent-ddp/fig3.png)

## 说点不好听的

它 0 star。十月二十三号建仓，现在共 114 次提交、五个版本，五个开着的 issue 全是依赖升级机器人提的。

iOS 这半边只支持模拟器，真机还没做。看不见的东西也列了一长串：SwiftUI Canvas 里的图文、系统弹窗、部分覆盖的文字、Compose 里某些画进图层的缓存绘制。

还有一类我特意试了：一个按钮只在手指抬起时才响应触摸，这种它判断不了，只会说触摸能到，然后 `tap` 之后告诉你「什么都没变」。

这不是缺陷，是它诚实。宁可说不知道，也不猜一个答案给你。

## 我的判断

这波工具真正的分水岭不是能读多少属性，是敢不敢闭嘴。

一个什么都敢报的检查器，agent 学会的是绕过它。一个只在该报的时候报的检查器，agent 才敢把它当验收条件写进流程。

九个真问题、2.7 秒、一千零九十一字节的输出，一个退出码。这套东西如果稳，agent 做 UI 就从「写代码」变成「写完自己验」。

你现在让 agent 改 UI，是发截图给它看，还是让它自己摸？
