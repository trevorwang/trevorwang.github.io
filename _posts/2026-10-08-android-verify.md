---
layout: post
title: "Android 侧载的苹果化只走了一半：要身份，不审内容"
date: 2026-10-08 08:14:49 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/android-verify/cover.png
---

![Android 侧载的苹果化只走了一半：要身份，不审内容](/assets/images/android-verify/cover.png)

如果你有朋友在巴西、印尼、新加坡或者泰国，9 月 30 日之后他可能会遇到一种新情况：从某个应用商店点安装，先弹一句开发者未验证，然后装不上。同一个时间点，你手上这台国行安卓机，装什么都不会有变化。

中文技术圈对这件事聊得不多，英文社区已经吵了一些日子。Hacker News 上 10 月 1 日那条帖子拿到 338 分、141 条评论，自由软件阵营的 F-Droid 联合电子前哨基金会（Electronic Frontier Foundation）发了公开信，标题就一句，反对 Android developer verification。

所以这条值得拆开看。它改了什么，谁真的会疼。

## 官方原话：应用要由已验证的开发者注册

Google 的说法很直接，[官方指南](https://developer.android.com/developer-verification/guides)第一句就是，Android 要求所有应用都由已验证的开发者注册，用户才能把它装进认证 Android 设备。

几个边界先划清楚：

- 生效时间 9 月 30 日，先落巴西、印尼、新加坡、泰国四个国家
- 覆盖七家商店，Google Play、HONOR 应用市场、OPPO 应用市场、三星 Galaxy Store、传音 Palm Store、vivo 应用商店、小米 GetApps
- 管的是 Android 7 以上的认证设备，检查通过 Google Play services 下发
- 这一轮只覆盖手机和平板两种形态
- 2027 年全球铺开，并把验证能力扩展到所有第三方 Android 商店

有个细节比表面重要：这轮只管上面这七家商店。官方 FAQ 写得很清楚，如果你从其他渠道分发，或者用户直接侧载，这些新要求暂时不管你。

也就是说，侧载的路没被堵，是入口处多了个证件查验。

![](/assets/images/android-verify/timeline.png)

Google 自己给的比喻也挺坦白，他们说这像机场查证件，和安检是两件事，确认的是开发者是谁，不审应用的内容，也不审它从哪儿来。

## 三条路，你得挑一条

官方给了三条路径，差别在身份验证这一栏。

| | 完整分发 | 有限分发 | 侧载未注册应用 |
|---|---|---|---|
| 适合谁 | 组织与职业开发者 | 学生、爱好者、个人 | 不愿验证的开发者 |
| 要不要身份验证 | 要 | 不要 | 不要 |
| 分发范围 | 任何商店或自有网站 | 最多 20 台设备 | 任何渠道，但用户要开 advanced flow |
| 用户体验 | 和今天一样 | 接受邀请后安装 | 一次设置，之后能装 |

完整分发要交身份信息，开发者控制台账号 25 美元一次，注册成组织还要 D-U-N-S 编号，申请最长要等 28 天，另外得有一个用 Google Search Console 验证过的官网。

有限分发是给不想交政府证件的人留的口子，免费，不限应用数量，但只能装到 20 台设备上，用户得接受你的邀请。学生、爱好者、自己写给自己用的人走这条最省事。

第三条路是给用户的：未注册开发者的应用仍然能装，但用户要先做一次 advanced flow 设置，官方说这些保障是给用户留出中断胁迫的时间。

有一处官方说法要留意，advanced flow 一旦关掉，未注册应用的更新就会失败。这条路只适合当终点，不适合当日常。

## 我拆了一个 APK，验证的对象其实就两样东西

讲概念不如自己跑一遍。我本机的 Android SDK 里有 aapt2、d8、zipalign、apksigner 和 keytool，够手搓一个 APK 了，没借助 Android Studio。

流程是：写一个 AndroidManifest.xml，里面只有一行包名 com.example.handmade，javac 编一个类，d8 转成 classes.dex，aapt2 link 生成 APK，最后用 keytool 现造一对 RSA 密钥，apksigner 签名。

签完把身份拆出来看：

```
Signer #1 certificate DN: CN=Handmade Identity Demo, O=none, C=CN
Signer #1 certificate SHA-256 digest:
0a220aaa9b719f34f97e88cb46a95de020fba812fee540251cc2cae7a188a269
Verified using v2 scheme (APK Signature Scheme v2): true

package: name='com.example.handmade' versionCode='1' versionName='1.0'
```

一个 8619 字节的 APK，身份就是这两行，包名 com.example.handmade，加证书指纹 0a22 开头那串 SHA-256。Google 要你登记的就是这一对东西，官方说法是提供用你私钥签名的 APK，把这个 APK 和你的开发者账号连起来。

![](/assets/images/android-verify/terminal.png)

再做一个对照就更有意思了。

我拿没签名的那个 APK 去验，直接报 DOES NOT VERIFY，缺 META-INF/MANIFEST.MF。然后我用第二把密钥签同一份 APK，代码一个字没改，包名也没变，证书指纹从 0a22 开头换成了 ee56 开头。

所以验证验的是人，不是代码。这也解释了官方 FAQ 为什么专门写一条，签名密钥丢了就没法注册包名。那把 keystore 文件不是构建缓存，是身份证，得跟隐私数据一样归档。

## 这不是 App Store 化，是 macOS 公证化

很多人第一反应是 Google 苹果化了。这个判断只说对了一半。

像的地方确实是「分发要挂账号」。苹果那边每个安装路径都挂在 Apple Developer Program 上，99 美元一年，组织同样要 D-U-N-S。macOS 上更直接，Gatekeeper 首次运行联网查一次公证票据，苹果在欧盟放开的第三方市场，也要求在安装时确认来源是授权渠道。

不像的地方才是关键，而且有两处。

第一，Google 不审内容。官方帮助文档里明确写了，不收集应用内容和功能信息。苹果那套是审查加抽成，内容规则一大堆，欧盟最新的条款里，走应用内购买抽 26%，走替代支付抽 20%，跳链接出去抽 15%，走第三方市场或网页分发再收 5%。

第二，Google 留了免身份的出口，20 台设备的有限分发、advanced flow、还有完全不受影响的 adb 开发安装，苹果那边没有对应物。

方向上还有个对照挺有意思。苹果在欧盟是被数字市场法按着往开走，Google 这边是主动往收。这不是谁学谁，是两条方向相反的曲线在中间碰头。

真正值得盯的是那句话，验证能力会扩展到所有第三方 Android 商店。到时候 Google 不只是自己院子的门卫，还是别人家商店的身份签发机关。

## 谁会疼

普通用户不会。体验真的没变，这也是反对者最难讲通的地方。

会疼的是另外三拨人。F-Droid 这样的非营利仓库首当其冲，他们的公开信里写得很直白：

> 在认证设备上安装或启动任何应用，都会回连 Google，验证开发者和应用是否已被批准。

他们托管几千个应用，验证按包名绑定，意味着每个志愿者都要把身份交给 Google。他们的结论是建议开发者不要注册。

第二拨是没法完成验证的人。官方 FAQ 自己承认，制裁地区的开发者无法完成验证。离线环境、拿不出证件、或者所在地区根本走不通流程的人，等于被排除在分发之外。

第三拨是未来任何想跟 Play 竞争的分发方，他们连上线都要先过 Google 的注册流程。

## 对国内开发者，三条实际动作

第一，如果你的应用只上 Google Play，而且用了 Play App Signing，你已经在这套流程里了，官方会自动认领，不需要额外操作。想确认状态，去 Play Console 的开发者验证页看一眼就行。

第二，如果你有海外用户、又走非 Play 渠道，现在就该做的事是把签名密钥归档管理起来，把 D-U-N-S 提前申请了，把 2027 当真正的截止日期，别盯着今年九月。

第三，只写给自己或小圈子用的应用，adb 这条官方保留，advanced flow 也能用，但记住更新未注册应用要靠它，别关掉。

还有个查询接口值得知道，Google 开了 Android Developer ID Status API，可以查某个包名有没有被注册，每天 1000 次。它要挂在 Google Cloud 项目上才能调，我本机没这个条件，所以只当线索记着。对做分发的人来说，它意味着装机前可以先问一句这个包名归谁。

## 最后

这不是一次安全检查，是分发的守门权从操作系统挪进了 Google 的账号体系。对普通用户几乎无感，对非主流分发生态是结构性打击。

2027 年全球铺开才是分水岭，现在这四国只是一个成本很低、随时可调的试点。

你手上要是有个只在国内分发的应用，会去注册吗，还是等 2027 再说？评论区聊聊。

参考：
- Android developer verification 官方指南 https://developer.android.com/developer-verification/guides
- 官方 FAQ https://developer.android.com/developer-verification/guides/faq
- F-Droid 公开信 https://f-droid.org/en/2026/02/24/open-letter-opposing-developer-verification.html
- Hacker News 讨论 https://news.ycombinator.com/item?id=49917761
