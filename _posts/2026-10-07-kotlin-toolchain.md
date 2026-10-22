---
layout: post
title: "Amper 转正了：它成了 Kotlin 的官方工具链"
date: 2026-10-07 11:52:28 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/kotlin-toolchain/cover.png
---

![Amper 转正了：它成了 Kotlin 的官方工具链](/assets/images/kotlin-toolchain/cover.png)

## 先说这周这条不太起眼的公告

10 月 6 日，JetBrains 的 Amper 博客发了最后一篇，标题是《Amper Is Now the Kotlin Toolchain》。名字后面还挂着一句 Moving to the Kotlin Blog，是字面意思，这个栏目从那天起不再更新，以后并到 Kotlin 博客里去。

改名的不是博客，是 Amper 自己。它现在叫 Kotlin Toolchain，是 Kotlin 的官方工具链。

这类新闻很容易被划过去，觉得又是改个名冲业绩。但我把它的代码和文档翻了一遍，发现这次不太一样：它是在把 Amper 这三个字从代码里一层一层往外清。

清到什么程度？看 0.13 的发布说明就明白了。破坏性变更里有两条，一条是把环境变量 AMPER_BUILD_DIR 改名成 KOTLIN_TOOLCHAIN_BUILD_DIR，另一条是把模块布局里那个叫 amper 的名字改成 default。

一个名字能从配置文件里清到环境变量里，说明这不是营销动作。

## 三个月里的三个动作

我按时间把这个过程排了一下：

![](/assets/images/kotlin-toolchain/timeline.png)

仓库搬家在 5 月，宣布转正在 6 月，把名字从代码里清干净是 10 月。前两个动作你在新闻里看得到，第三个动作只有翻开 0.13 的发布说明才看得到。

6 月那篇里有句话我挺喜欢，它解释了为什么要换名字：

```text
As the project evolved, it became clear that the ecosystem doesn't just need
another build tool – it needs a unified entry point into all of Kotlin.
```

> 译文：随着项目往前走，我们看清了一件事，这个生态要的不是又一个构建工具，而是一个通往整个 Kotlin 的统一入口。

## 我在这台机器上装了一遍

不看文档里的说法，看它实际怎么跑。装法是官方给的脚本，我没直接管道给 shell，先拉下来读了一遍，脚本里带 SHA256 校验，做题还算规矩。

```text
curl -fsSL -o /tmp/kt-install.sh https://kotl.in/install.sh
```

它默认会往你的 shell 配置里追加一行 PATH，我用它自己的开关跳过了，不碰我的 zshrc：

```text
KOTLIN_CLI_NO_MODIFY_PATH=1 sh /tmp/kt-install.sh
```

装完是这样：

```text
Kotlin Toolchain version 0.13.0 (abc7a5f, 2026-10-01)
Installed Kotlin CLI to /Users/trevorwang/.local/bin/kotlin
```

然后建项目。它有一批非交互的模板，我选了 JVM 命令行那个：

```text
kotlin init --from-template=jvm-cli --no-git
```

0.34 秒生成完。我以为要看一堆文件，结果整个项目的配置文件就一行：

```yaml
product: jvm/app
```

源码三个文件：main.kt 里一句 println，World.kt 里一个 object，test 目录里两个测试。

## 构建、运行、测试全跑了一遍

第一次构建它自己下了一个 Temurin JDK 25，下完抬头就是绿的。装、建、编、跑、测连着走一遍是这样：

![](/assets/images/kotlin-toolchain/terminal.png)

整个过程 30.8 秒，扣掉下 JDK 的时间，剩下的很快。测试那段是红的，因为模板里故意放了一个 assertTrue(false)，函数名就叫 shouldFail。我猜它是想让你先看一眼失败的输出长什么样，顺带告诉你退出码是 1。

还有个命令我挺喜欢，kotlin show settings。它把每个模块生效的设置打出来，默认值后面标一行 # default：

```text
compose:
  version: 1.12.1  # default
junit: junit-5  # default
```

哪些是你自己写的，哪些是它默认给的，一眼分得清。

## 三行 YAML 编四个平台

单个 JVM 不够看，我又用 kmp/lib 模板建了一个。它给我的 module.yaml 全文，加上各平台编出来的结果是这样：

![](/assets/images/kotlin-toolchain/yaml-platforms.png)

平台专属的源码不用 Groovy 那套 source set 命名，是目录后缀：src@jvm、src@android、src@ios，里面各放一份 actual。共享的那份 expect 放在 src。

构建的时候还能只挑平台：

```text
kotlin build -p jvm -p android
```

11.3 秒，四行绿字，产物是 kt-kmp-jvm.jar 和 kt-kmp-jvm.aar。

这里有个我事先没想到的细节。翻它下载缓存的目录时我发现，为了编 Android，它往我的 Android SDK 里放了一个平台包，时间戳跟这次构建对得上：

```text
~/Library/Android/sdk/platforms/android-37.0/            11:35
~/Library/Android/sdk/platforms/06259a9c8a-platform-37.0_r02.zip.flag
```

加上前面那个 Temurin JDK、测试用的 JUnit 运行器、iOS 要用的 Kotlin/Native，全是它自己拉下来的。对新人友好到有点过分，装完一个命令，什么都不配，JVM 和 Android 就能编。代价是第一次构建得等下载，而且它会往你的 SDK 目录里放东西。

## iOS 那条腿，它停在了 Xcode 上

把 iOS 也加上试试：

```text
kotlin build -p iosArm64
Downloading .../kotlin-native-prebuilt-2.4.20-macos-aarch64.tar.gz
ERROR: Xcode first-launch setup could not be completed.
       Run `sudo xcodebuild -runFirstLaunch` and try again.
ERROR: Task 'xcodeEnvironment' failed: Xcode environment is not ready
```

它先把 Kotlin/Native 2.4.20 下下来，然后停在这一步。这个检查是 0.13 新加的，发布说明里写的是 Check and/or provision Xcode components automatically。

有意思的地方在这：Kotlin/Native 本身在我这台机器上是能编的，之前我拿它直编过 iOS arm64 的 framework，9.8 秒出产物。工具链检查的是 Xcode 那套组件有没有完成首次启动安装，比编译器本身的要求更靠前，而这一步要 root，得自己跑一次带 sudo 的命令。

这个行为我觉得是好的。它没有编到一半丢给你一个莫名其妙的链接错误，而是停在前置检查上，把该敲的命令写给你。

## 名字还没清干净的两个地方

第一处，它下载发行包的地址还是老地址，里面带着 amper 两个字，packages.jetbrains.team/maven/p/amper/amper。

第二处更逗，Android 那边的修复清单里有一条，说它以前会把 Android SDK 装成非规范的名字，那个名字叫 Amper-33。

仓库、博客、命令行、环境变量、模块布局，一路清过来，边角还没扫完。这种细节比新闻稿实在，你从里面能看出一次改名到底动了几层。

## 界面那一侧，多了个开关

命令行那套说完了，界面这一侧的变化同样明显。

插件在 Marketplace 里就叫 Kotlin Toolchain，装的版本是 262.10968.205。它的自我介绍第一句是：

```text
Provides support for Kotlin Toolchain, JetBrains' build tool for Kotlin and
Java projects.
```

> 译文：为 Kotlin Toolchain 提供支持，也就是 JetBrains 那套面向 Kotlin 和 Java 项目的构建工具。

![](/assets/images/kotlin-toolchain/plugin-marketplace.png)

功能列表里有三件我关心的：跑和调 JVM、Kotlin Multiplatform、native 的测试，管多模块工程和版本目录，还有 Compose Desktop 与 Compose Multiplatform 的热重载。页面顶部另有一行提示，说部分功能要 Ultimate 订阅才有。

真正有意思的是新建项目那个对话框。Build system 那一栏不再是 Gradle 一个选项，变成了两个并列，右边那个就是 Kotlin Toolchain。

它给这两条各写了一句说明，原话是这样：

```text
Gradle — Flexible and backed by a mature plugin ecosystem
Kotlin Toolchain — Easy to use, declarative, and AI-friendly
```

> 译文：Gradle 灵活，背后有成熟的插件生态；Kotlin Toolchain 好用、声明式，对 AI 友好。

![](/assets/images/kotlin-toolchain/new-project.png)

AI-friendly 这个词是它自己写上去的。一家做 IDE 的公司在新建项目的默认文案里把「对 AI 友好」当卖点，放在今年不算意外，但这个位置挺说明问题。

勾平台那部分跟 CLI 的模板一个思路，Android、iOS、Desktop、Web、Server 五个复选框，iOS 和 Web 还能选 UI 要不要跟其他端共享。勾完 Android 加 iOS 建出来，结构是这样：

![](/assets/images/kotlin-toolchain/project-structure.png)

根目录一份 project.yaml 列模块，androidApp、desktopApp、iosApp、shared、webApp、core、server 七个。每个模块自己一份 module.yaml，跟刚才 CLI 里那份三行的配置是同一个东西。版本号仍然放在 libs.versions.toml 里，Gradle 那套习惯它没丢，wrapper 脚本叫 kotlin 和 kotlin.bat。

运行配置那边也能直接用，设备栏里挑 iPhone 17 | iOS 27.0 就能跑 iosApp。

## 那你要不要换

官方 FAQ 里有一段，直接回答为什么不去改 Gradle，我把原话贴出来：

```text
With the Kotlin Toolchain, we want to show you our design and get your feedback,
as it will help us to decide which direction to take the design. At the same time,
we are also working with the Gradle team to improve Gradle support in our IDEs
and Gradle itself.
```

> 译文：我们想把设计摆出来给你看，靠反馈决定往哪走。同样的时间，我们也在和 Gradle 团队一起改进 IDE 里的 Gradle 体验，以及 Gradle 本身。

话说得客气，意思很清楚：这不是要把 Gradle 换掉，是一次同台展示。

迁移那件事也有意思。Maven 项目给了转换命令 kotlin tool convert-project，能自动转一部分。Gradle 项目没有工具，官方解释是 Gradle 脚本本身是任意代码，写不出确定性的转换规则。

那怎么办？FAQ 里有一句让我停下来的话，大意是这种事 AI agent 挺擅长，配一套技能就行。我去看了那套技能，在 Kotlin 组织底下，叫 kotlin-agent-skills，1,071 星，十个技能文件，其中两个专门讲怎么从 Gradle 迁到 Kotlin Toolchain。

里面规矩写得很硬：每一个第三方 Gradle 插件只有三条路，砍掉、换成原生等价物、自己重写，不存在留着继续用的选项；而你自己写的 buildSrc 那套约定插件不算插件，算配置，要变成模块模板。

这句话值得抄在本子上。把构建脚本当代码看，AI 能帮你做机械翻译；把构建脚本当配置看，那套隐式约定才是迁移里最贵的东西。

## 收尾

这周的动作概括一下：Amper 这个名字退休，它留下的东西成了 Kotlin 的官方工具链，门面是一个 kotlin 命令加一份 YAML。

新项目值得试，三行配置编四个平台，这套体验确实舒服。老项目先别动，它还是 Alpha，官方自己都写了不保证所有迁移场景。

仓库和文档在这：

```text
github.com/JetBrains/kotlin-toolchain
kotlin-toolchain.org
```

最后留个问题：如果让你现在把手上的 KMP 项目从 Gradle 迁过来，你猜最先卡住你的是哪一步？
