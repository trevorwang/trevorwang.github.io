---
title: "Kotlin 2.5 换了发布格式，编译器也跟着变严"
date: 2026-10-22T07:33:31+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin"]
summary: "10 月 21 日，Kotlin 2.5.0-Beta2 发布。多数人翻开更新说明，看的是语言特性那几行：四项转正、两项实验。看完觉得这一版平平无奇，就关掉了。"
cover:
    image: "/images/kotlin25/cover.png"
    alt: "Kotlin 2.5 换了发布格式，编译器也跟着变严"
    relative: false
---

10 月 21 日，Kotlin 2.5.0-Beta2 发布。多数人翻开更新说明，看的是语言特性那几行：四项转正、两项实验。看完觉得这一版平平无奇，就关掉了。

真正的变化在后面几节。发布格式换了，编译方案换了，还有三处默认值被悄悄调严。我把两个编译器装在同一台机器上，同一份源码各喂一遍，出来的结果不一样。

## 这次发布，重点不在语法

四项稳定特性里，有两项其实是「把选择变成默认」：按名解构的 only-syntax 模式、未使用返回值检查器。老代码一行不用改，新代码的表达力会变。

实验性的两项也在同一路数上，companion 扩展与 companion 块、完整 value class。都是让语言少一点莫名其妙的限制。

语言层这次最安静。真正铺开来看，是文档后半段那三节：Kotlin Multiplatform 的发布格式、新的编译方案、Apple 平台的增量编译。这三条不影响你写什么，影响你交付什么、多久能编完。

## 一个 .kar 文件，替掉七千多个文件

官方的原话是这样：

> The new format publishes platform artifacts (except for JVM and Android) as a single .kar archive, which is a ZIP archive additionally compressed with XZ.

一个库支持几个目标，过去就要发几套产物。现在除 JVM 和 Android 之外的平台产物，全打成一份 `.kar`，本质是 ZIP 再套一层 XZ 压缩。

数字比描述直观。官方拿 cryptography-kotlin 举例：发布产物从 7539 个文件、91 MB，变成 1036 个文件、23 MB。整个多模块工程。

![官方发布说明原文，红框是这组数字](/images/kotlin25/fig1_kar.png)

开启方式一行配置，单个模块写在构建脚本里，全工程写在 `gradle.properties`：

```
kotlin.publicationFormat=KOTLIN_ARCHIVE
```

两个限定必须一起看。除 JVM 和 Android，这两个生态仍旧走标准的 Maven 产物。消费侧已经全面可用，你依赖的库将来换格式，你的构建不用动。发布侧还是实验特性，你自己要发库的话，得单独验一遍流水线。

## 为什么 Maven Central 卡的是文件数

原文只说了一句「Maven Central 最近引入了发布体积限制」。这句话背后的事，比体积有意思。

卡住库作者的不是总字节数，是文件数。一个支持六个目标的库，每个目标一套产物，再算上源码包和元数据，随便就上几千个文件。每个文件在仓库里是一条独立记录、一次上传、一次索引，对仓库是运维成本，对使用者是依赖解析时第一屏的请求数。

所以文件数从 7539 掉到 1036，降了百分之八十六，比体积从 91 MB 掉到 23 MB 更关键。省下来的是请求和索引，不是带宽。

JVM 和 Android 被排除在外，也说明一件事：这两个生态还锚在标准 Maven 分类器上，动不了。跨平台库的真实处境是「多目标，主流平台守老格式，其余走归档」，这个混合结构会持续一阵子。

这条改动对下游完全透明，正因为透明，它才敢被推荐给所有人。受益的是库作者和维护发布流水线的人，不是写业务代码的你我。

## 新的编译方案：common 代码不该偷偷用上平台 API

这一节官方写得克制，四条：让编译器处理 common source sets 的方式与 IDE 一致；防止 common 代码意外解析到平台特有声明；改进重载解析与类型推断的一致性；为 common source set 启用增量编译。

前三条在修一个老坑。`commonMain` 里写错一个 import，IDE 标红但命令行能过，或者反过来，两边都对不上。原因是 IDE 和编译器各有一套解析器，各算各的。

统一之后有个副作用：以前能编过的代码，现在可能编不过了。这不是退步，那些代码本来就摸到了不该看见的符号，只是没人拦。

第四条才是日常收益。common 终于有增量编译了，天天开 IDE 的人感受最直接。CI 上反而看不出来，因为流水线每次都是全量构建。

JetBrains 在九月专门为这个方案写过一篇博客，标题里用了 reliable 这个词。要修的问题正好就是「不可靠」：同一份 common 代码，IDE 和编译器给出两种结论。

## 编译器变严了，三处默认值都改了

先看最直观的一处。同一份代码，2.4.21 直接编不过，2.5.0-Beta2 默认就过：

```
(val mail = email, val name = username) = user
```

括号里带 `val`，用等号指定取哪个属性，变量名不用和属性同名。

![三个特性的本机实录](/images/kotlin25/fig3.png)

顺带说个老坑。`val (email, username) = user` 这种按位写法两个版本都通过，但值是按位置分的。我把 `User(username, email)` 解成 `email=alice`、`username=alice@example.com`，变量名全错，编译器一声不吭。这个特性存在的理由就是它。

第二处是未使用返回值检查器。同一个文件，2.4.21 静默通过，2.5.0-Beta2 直接给 warning：`unused return value of 'plus'`。表达式算出了非 Unit 的值又没人用，它会指出来。

第三处是跨模块内联。2.4.0 引入时是可选的，2.5.0-Beta2 默认开启，回退开关是 `-Xklib-ir-inliner=disabled`。

另外还有一处不在默认值上：类型推断的约束生成减少了，处理复杂泛型时不再卡住。官方写得很克制，只说了可能改变推断结果、重载选择、诊断。翻译成人话是，升级后某些代码会推断出不同的类型，而那些代码你之前认为是对的。回退开关 `-XXLanguage:-EliminateSecondKindIncorporation`。

## value class 这一刀切得最深

值类以前只允许一个属性，还得配 `@JvmInline`。现在多属性可以，另外多了抽象 value class 和 value object 两种形态。

本机实测：2.4.21 对多属性 value class 直接报错，提示要开 `-XXLanguage:+FullValueClasses`；2.5.0-Beta2 加上 `-Xfull-value-classes` 就通过。

value object 同理，2.4.21 说 `modifier 'value' is not applicable to 'standalone object'`，2.5 加上开关能编能跑。

有个细节比文档更能说明问题。value object 上写 `===`，编译器直接报错，identity 比较被语言禁掉了，因为它本来就没有身份。

真正的坑在迁移。`value class` 不再默认 inline。官方给的时间表是 2.5.20 开始出 warning，2.6 或 2.7 变 error。2.5.0-Beta2 上这个开关还认不出来，编译器回的是 flag 不支持，说明窗口确实要等到 2.5.20 才开。

也就是说现在升级不会炸。但你要在 2.5.20 之前把老代码标清楚，决定哪些继续 inline、哪些放手。一个只包一个 String 的值类到底该不该 inline，本来就是取舍，语言不该替所有人做这个默认。

## 标准库补了两个小口子

`if` 表达式这块补了三个函数：`onTrue()`、`onFalse()`、`ifOrNull()`。以前写「添加成功了就打印一句」，得先存个局部变量再套 if/else，现在一行接住：

```
val added = tags.add("wasm").onTrue { println("Tag added") }
```

返回值还是原来那个布尔，判断和副作用挤在一条链上，不用再为这种小场景单独造变量。

另一半是 Unicode 码点。`Char` 只装得下 UTF-16 的一个代码单元，碰上 emoji 这种代理对，按 `Char` 遍历会把一个字符拆成两半。新的 `CodePoint` 是个 value class，底层用 Int 存码点，能查它占几个 UTF-16 单元、属于哪个类别、是不是字母。

这两处都要 `@OptIn(ExperimentalStdlibApi::class)`。留意一下 `CodePoint` 的形态：它本身就是 value class 放宽之后才写得出来的东西，语言和标准库在这一版里对上了。

## 我在本机看到的两个意外

![同一份代码，两个编译器的对照](/images/kotlin25/fig2.png)

第一个意外是编译没有变快。489 行的工程，2.4.21 预热后中位数 2.86 秒，2.5.0-Beta2 是 2.99 秒，产物都是 161 KB。差异落在噪声区间，不能说变慢，但也谈不上推断优化在小工程上有什么收益。这类优化只在大型泛型代码里才显形，小样本看不出来。

第二个是开关的适用范围。`-Xklib-ir-inliner=disabled` 在 JVM 编译器上直接不被接受，提示 flag 不支持。它只在编 klib 的平台有意义，也就是 Native、JS、Wasm 那几路。

文档里写的「默认开启」，按平台分开读才对。

## 升级前该做的三件事

JDK 门槛抬了。编译器的最低要求从 8 提到 17，现在低于这个版本直接报 error。CI 镜像和 Docker 基础镜像要跟着升，这一步通常比代码改动更耗时间。

发布 Kotlin Multiplatform 库的话，`.kar` 先别急着开。发布侧还是实验特性，可以在流水线里试跑，别直接推正式版。

测试要跑一遍。未使用返回值检查和类型推断的变化，都属于静默通过但结果不同的那类，读文档看不出问题，跑测试才看得见。

## 最后

2.5 这一轮的动作大多落在默认值上：默认打包、默认检查、默认内联，还有默认按名字解构。每改一处都配了回退开关，说明官方清楚会踩到人，也说明这些默认值还没定稿。

你手上有没有那种「IDE 标红、命令行却能编过」的代码？升级到 2.5 之后它大概得老实交代。欢迎在评论区聊聊你踩到的变化。

参考资料：
[Kotlin 2.5.0-Beta2 更新说明](https://kotlinlang.org/docs/whatsnew-eap.html)
[Kotlin 版本发布计划](https://kotlinlang.org/docs/releases.html)
[Kotlin Multiplatform 新编译方案](https://blog.jetbrains.com/kotlin/2026/09/a-more-reliable-compilation-scheme-for-kotlin-multiplatform-modules/)
