---
title: "AI 都能写 Swift 和 Kotlin 了，跨端图什么"
date: 2026-10-23T14:31:26+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "Flutter"]
summary: "9 月 10 日，Shopify 移动端负责人 Mustafa Ali 发了篇博客。"
cover:
    image: "/images/flutter-ai-scale/cover.png"
    alt: "AI 都能写 Swift 和 Kotlin 了，跨端图什么"
    relative: false
---

9 月 10 日，Shopify 移动端负责人 Mustafa Ali 发了篇博客。

标题很直白，原生现在是 Shopify 移动端的未来。

六年前他们 all-in React Native。去年 1 月，Ali 还写过复盘，说 RN 前途光明。现在收回去，理由一条，coding agent 把写两遍的成本打下来了。

转过去的效果不差。Shop App 六个工程师，12 周从概念验证做到上架。iOS 冷启动快 23%，Android 快 50%。

![Shopify 官方博客里的迁移示意图](/images/flutter-ai-scale/shopify-native.png)

但这段话里最值钱的不是数字。

Ali 自己写，agent 几秒能改完代码，却要花好几分钟才测得出结果。因为模拟器控制太慢，只能靠无障碍树或截图去拿状态。

他的原话是，如果模型没法快速验证自己的工作，模型多好都不管用。

这句抱怨，让我把两篇东西并排读了一遍。Shopify 9 月 10 日这篇，和 Flutter 团队四十天后的回应，争的其实是同一件事。

## 瓶颈不在生成，在验证

Flutter 团队那篇 10 月 20 日发，作者是 Carlos Mendonça 和 Emma Twersky。标题也直白，AI 让迁移变简单，Flutter 让扩张变简单。它要回答的，就是 Shopify 抛出来的那个问题。

全文压在一个判断上，代码生成早就不是瓶颈，验证才是。

两个数据撑着这个判断。73% 的开发者说用 AI 开始任务时更有产出。46% 的人不信任 AI 在没有人工检查时的调试准确性。

两头一夹，卡点在人这边。

## 三个环节的账

官方给了张对照表，把原生和 Flutter 放在一起比验证的三个环节。

![按官方原表重排的中文对照，三行数值与表述未改动](/images/flutter-ai-scale/verify-table.png)

代码到运行的延迟，原生 2 到 5 分钟一次。构建、链接、归档、装模拟器，四轮调试就耗掉 20 多分钟。

Flutter 200 毫秒以内。热重载把改动注入运行中的 Dart VM，状态还留着。

自动化测试，原生要走完整的模拟器生命周期和后台 emulator。Flutter 的 headless widget test 在隔离环境里跑，500 毫秒以内。

界面和状态检查这条最见功夫。

原生截模拟器图喂给视觉模型猜布局，概率性的、慢。Flutter 走 MCP 查 Dart VM Service 和 DevTools，直接读 Widget 树和 RenderObject 树，确定的、结构化的。

三段串起来是一条完整的回路。原生那边每走一段都得排队，Flutter 这边一个来回就几毫秒。

## harness 比 prompt 值钱

验证快是一回事，怎么让 agent 用上它，是另一回事。官方给了个内部实验，Project One Shot，让 agent 自主修 camera_android_camerax 里的生产 bug。

这插件管着 Android CameraX 的生命周期和 surface texture 绑定，Flutter 里最复杂的那类代码。

给泛泛的指令，比如注意别内存泄漏，agent 全失败。

换成确定性的 harness 才成。AST 分析走 MCP、包级别的 domain skill、强制的测试驱动 guardrails，先用一个失败的测试复现，再写最小修复，收尾跑通全套测试和泄漏检查。

就是不给 agent 自由发挥的空间，把每一步的验收条件钉死。

配套放出三样东西。官方 Dart 和 Flutter agent 插件，把 MCP server 和 10 个 Flutter、14 个 Dart 技能打包，agent 能直接访问 Dart Analyzer 和 Widget Inspector。

Antigravity 里的专用 agent，自动审无障碍问题并提交符合习惯的修复。还有个和 Serverpod 一起做的 Agent Skills CLI，一条 dart run skills@get 把依赖包带的 skill 装进 agent。

基准数据是 Gemini 3.5 Flash 从 84.1% 到 85.5%，Gemini 3.1 Pro 从 89.6% 到 92.7%。涨幅不大，是官方自己拿评估套件跑出来的。

## 这些能力落到什么地方

官方那篇列了一串落地位置，比手机远得多。丰田和雷克萨斯的车机、小米 SU7 的中控、Nubank 一亿用户的金融 App、Universal 音乐。

文章没往这个方向推。我倒觉得这是它最有说服力的一段。

这些场景的共同点是验证成本高得离谱。

车机刷一次固件几十分钟。金融 App 一次线上事故就是真金白银。音乐 App 的用户对卡顿零容忍。

在这种地方，验证回路短一分钟，都是实打实的钱和时间。

再往远看一点。agent 写代码变成常态之后，一份代码跑多端省的不只是人力，还有时间。

agent 生成一版，热重载到手机、桌面、Web 同时看效果，比一个平台一个平台地编译、打包、装模拟器快一个数量级。

这也是 Flutter 团队这两年往车机、桌面、嵌入式使劲的原因。

跨端的下半场，可能不在手机里。

## 有几点我不是很同意

应用商店评分那段其实经不起推敲。原文引了 Apptopia 对 201,000 个应用的统计。Google Play 上 Flutter 应用平均 4.1 星，非 Flutter 也是 4.1 星。App Store 上都是 4.0 星，结论是用户不关心框架。

这数据只能证明跨端没拖累评分。翻过来也说明它没带来优势。

拿一个打平的结果去论证应该选它，差着一层。

热重载那个 200 毫秒，说的是注入改动的时间，不含改代码、想清楚、切窗口。

它也改不了类的结构。加字段、改状态树还得重启。

官方表里比的是代码到运行的延迟，这个定义是诚实的。但读者容易读成开发速度是原生 20 倍，两件事差得远。

企业案例也有幸存者偏差。

小米 SU7、Universal 砍掉 45% 代码、丰田和雷克萨斯、Nubank 一亿用户，都是真的。

但没有一篇文章会写某公司迁到跨端两年后又拆回去。拿这些论证架构选择，只能算半个证据。

![官方给出的八条论点，拿去团队里吵架够用](/images/flutter-ai-scale/talking-points.png)

## 跨端的账该怎么算

先把两家的账摆平。Shopify 的判断是，写两遍的成本被 agent 打下来了，剩下那点不值得为了省它放弃原生能力。Flutter 的判断是，协调成本还在。

原文举的例子很具体。同一个结账状态机，同一个模型开两个分身，一个写 Swift 用异步 stream 丢过期事件，一个写 Kotlin 用 reactive flow 把事件排队。

两套实现第一天都过了本地单测。几周后上线，Android 结账转化率掉三个点，原因是网络抖动时重试时序不一样。

![官方原图：并行 agent 不会消除协调成本，而是引入非确定性 drift](/images/flutter-ai-scale/parallel-agent-challenge.png)

两种说法都成立，算的是不同的账。

Shopify 这次转向，我也不觉得该当成"跨端已死"的证据。

它是几百个屏幕、上千工程师、把移动端当核心业务在投的公司。这个体量下为了 23% 的冷启动和崩溃率的改善去重建，账算得过来。

换成五个人维护一个中等规模的 App，同样的事就未必划算。

决定你要不要跨端的，是团队规模、维护年限，还有验证链路的长度。这几样 AI 一个都改不了。

倒是另一个问题值得吵一吵，如果 agent 真能把一轮验证压到几百毫秒，那原生体验更好这条理由，还能撑多久。

想听你们手上的真实情况。你的项目现在是单端、双端还是跨端？当初选它，主导因素是性能、人力，还是那句老板说要原生？

顺带问一句，你觉得 Compose Multiplatform 有戏么？同样是跨端，它比 Flutter 晚进场好几年，但背后站着 Kotlin 和 Android 那半个生态。

评论区说两句，我挑几个有代表性的整理成下篇。

两篇原文都在这儿，加起来二十分钟。

[Native is now the future of mobile at Shopify](https://shopify.engineering/back-to-native)

[AI makes it easy to migrate. Flutter makes it easy to scale.](https://flutter.dev/blog/ai-makes-it-easy-to-migrate-flutter-makes-it-easy-to-scale)
