---
title: "eMudhra 证书链出事，我把 JetBrains 全家体检了"
date: 2026-11-03T09:10:00+08:00
draft: false
categories: ["公众号"]
tags: ['JetBrains', '证书', 'macOS']
summary: "八张中间证书私钥可还原，四张根证书要拉黑。跟着 JetBrains 公告自查之后我发现：官方 workaround 只删两把，我机器上四把都在，三个 JVM 一个都没修。"
cover:
    image: "/images/emudhra/cover.png"
    alt: "eMudhra 证书链出事，我把 JetBrains 全家体检了"
    relative: false
---

十月十七号，一个安全研究员报告：eMudhra 签发的八张中间证书，私钥可以还原出来，因为它们生成时用的熵有问题。

这意味着任何能插你网络的人，都能拿这几把私钥签一张「看起来可信」的证书，冒充 JetBrains 的工具要连的服务。

十月三十号 JetBrains 发了两篇公告。一篇讲 IDE 怎么办，一篇是全线产品的状态表。我看了一圈中文转载，基本都是转述，没人真去机器上查。

这篇就是那份自查手册。全程本机命令，不改任何配置。

## 官方到底说了什么

受影响的是用 Java 跑的东西。IDE、Toolbox、Gateway、ReSharper，因为它们信任的是随包分发的 JetBrains Runtime 的信任库，不是你 macOS 系统的钥匙串。

公告里写得很直白：更新操作系统对这一类问题没有保护作用。

修复版 JBR 不再信任这四张根证书：emSign ECC Root CA G3、emSign Root CA G1、emSign Root CA G2、AAA Certificate Services。

拆开看是四个 CVE：CVE-2026-91403 和 CVE-2026-114820 是 Oracle 在 JBR 里做的修复，CVE-2026-118696 是内置浏览器 JCEF 拿 Chrome 的证书更新列表，CVE-2026-118697 是后台服务 jetbrainsd 自己那份内置清单。

三层，各自修。

![一条事件，三层修复：JBR、JCEF、jetbrainsd](/images/emudhra/fig1.png)

## 第一步：先看版本对不对得上

别急着删证书，先确认你装的是哪个版本。JetBrains 的 `.app` 里有个 Info.plist，构建号就在里面。

```
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' \
  "/Applications/Android Studio.app/Contents/Info.plist"
/usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' \
  "/Applications/Android Studio.app/Contents/Info.plist"
```

把路径换成你装的 IDE 就行，Toolbox 装的在 `~/Applications` 下面。

我机器上的结果：

- IntelliJ IDEA 2026.2.3，构建 IU-262.10968.63，打包的 JBR 是 25.0.4+1-508.27
- Android Studio 2026.2，构建 AI-262.9437.185.2621.16467767，JBR 25.0.3
- JetBrains Toolbox 3.9.1，构建 3.9.1.89198

对照公告的修复列表：IDEA 要 2026.2.4 才算修，我这是 2026.2.3，差一个补丁。Toolbox 3.9.1 在修复名单里，这项过了。

Android Studio 是个空白。它不在 JetBrains 那张表里，Google 侧我没找到对应公告，只能自己看 JBR 版本：25.0.3，而修复的 JBR 25 是 635.79、508.41、329.142 这几个构建。25.0.3 显然是老的。

![四个组件，谁修了谁没修](/images/emudhra/fig2.png)

## 第二步：直接问信任库还剩几把

版本号会骗人，信任库不会。每个 JBR 自带一份 cacerts，用它的 keytool 就能列。

```
"/Applications/Android Studio.app/Contents/jbr/Contents/Home/bin/keytool" \
  -list -cacerts -storepass changeit | grep -iE "emsign|comodo"
```

Android Studio 里出来四行：

- `comodoaaaca [jdk]`
- `emsigneccrootcag3 [jdk]`
- `emsignrootcag1 [jdk]`
- `emsignrootcag2 [jdk]`

四张全在。IntelliJ IDEA 那份也一样，四张全在，证书总数 111。

这里有个坑我必须先说，因为我踩了：别名的字面带一个空格加方括号的后缀，`emsigneccrootcag3 [jdk]`。你只写 `emsigneccrootcag3` 去删，keytool 会告诉你这个别名不存在，但它明明就在列表里。

## 第三步：证明你的处置真的生效

这是官方文档里没有的一步。

官方的临时方案是：把 cacerts 复制一份，删掉受影响的根，然后用 VM option 把 IDE 指向这份副本。问题是，你怎么知道那个 `-Djavax.net.ssl.trustStore` 真的被读进去了？路径写错、少写一个斜杠、用了 `~`，它都会静默失效。

我写了个三十行的小程序，直接问 JVM：你现在到底还信几张。

![同一个信任库，三种状态：未修、官方 workaround、全删](/images/emudhra/fig3.png)

关键在于 `tmf.init((KeyStore) null)` 这一句，它让 JVM 用默认信任库，也就是会响应 `javax.net.ssl.trustStore` 的那个。然后数一遍 `getAcceptedIssuers()` 里有多少张证书的颁发者叫 emSign 或者 AAA Certificate Services。

跑出来是这样：

不带 VM option 时，它报 109 张、命中 4。指向官方清理版，107 张、命中 2。四把全删，105 张、命中 0。

三行数字，把「我以为我修了」和「确实修了」分开了。

## 官方没写的两件事

第一件，官方 workaround 只删两把：`comodoaaaca [jdk]` 和 `emsigneccrootcag3 [jdk]`。

但我在机器上查到的是四把都在，G1 和 G2 也还在信任库里。删完那两把之后，我的探针仍然报告「命中 2」。也就是说，按官方那两步走，这台机器上还留着两张 eMudhra 根证书。

我没有立场替 JetBrains 解释为什么只列两把，可能是修复版运行时对 G1、G2 另做了处理，也可能只是保守。但就「机器现状」这件事，数字是 4 不是 2。你要更彻底，就把四张一起删，我实测删完是 105 张、命中 0。

第二件，macOS 自己的系统根库里，这两张还躺着：

`emSign ECC Root CA - G3` 的 SHA-1 是 3043FA4FF257DCA0C380EE2E58EA78B23FE6BBC1，`emSign Root CA - G1` 是 8AC7AD8F73AC4EC1B5754DA540F4FCCF7CB58E8C。

`security dump-trust-settings` 在系统和用户两个域里都没有给它们单独加「不信任」策略。这不等于 macOS 就没拦它们，TLS 层还有别的机制，但至少从钥匙串这一侧看，它们还在。

## 顺手把机器上其他 JVM 也扫一遍

这一条是我觉得最容易被漏掉的。

JetBrains 只说了自家产品。可你的 Gradle 守护进程跑在某个 JDK 上，命令行编译 Kotlin 也用着某个 JDK，Homebrew 又装了一份，每一个都有自己的 cacerts，跟 IDE 那份一点关系没有。

我把机器上所有能跑 keytool 的 JVM 都过了一遍：

- Homebrew OpenJDK 21.0.10，109 张，四把全在
- Android Studio 打包 JBR 25.0.3，109 张，四把全在
- IntelliJ IDEA 打包 JBR 25.0.4，111 张，四把全在

一个都没修。

JetBrains 的修复版 JBR 也给到了 jbr21 的两条线，21.0.11b1163.125 和 21.0.9b1038.91。上游 OpenJDK 什么时候跟上，我没有一手出处，不能替你下结论，只能说本机这把 21.0.10 里四张还在。

## 该怎么做

优先级很清楚，能升级就别折腾临时方案。

IDEA 2026.2.4 已经在十月三十号发了，我从 JetBrains 的发布接口核过日期。Toolbox 3.9.1 已经装好，jetbrainsd 我也查了，跑的是 0.9.9728，就是修复版本。所以剩下要动的只有 IDE 本身。

如果暂时不能升，官方的三步是：复制 cacerts、用 jbr 里的 keytool 删别名、在 Help 的 Edit Custom VM Options 里加一行 `-Djavax.net.ssl.trustStore=` 指向副本。

路径必须写全，不许带 `~`，不许带引号。写错它不会报错，只会静默失效。然后在 Settings 的 Server Certificates 里确认「自动接受非可信证书」是关的。

还有一个细节：内置浏览器 JCEF 那份证书列表改不了，2025.2.6.4 及更早版本要直接关掉它，注册表键是 `ide.browser.jcef.enabled`。关掉之后 Markdown 预览和 notebook 输出会受影响，升级回修复版记得打开。

![三条处置路径：升级、临时方案、只修后台服务](/images/emudhra/fig4.png)

## 我的判断

这件事最值得记的不是哪个版本修了，是信任边界在哪。

你更新了 macOS，你以为你安全了。你更新了 Chrome，你以为你安全了。但 IDE 里面有一份自己的信任库，Gradle 又有一份，命令行那把 JDK 还有一份。每一份都要单独修。

安全公告是写给所有人的，它不会替你回答「我这台机器现在到底是什么状态」。那个答案只能自己跑出来。

你机器上现在有几个 JVM？扫一下吧，我猜不止一个还剩着那四张。
