---
title: "交完 25 美元才知道，Google Play 给自己装应用不要钱"
date: 2026-10-26T09:31:22+08:00
draft: false
categories: ["公众号"]
tags: ["Android"]
summary: "十月我为了一个开发者账号忙了小半个月。25 美元报名费，身份验证要政府签发的证件，付款资料的地址必须和证件上一字不差，新注册的个人号还得先找 12 个测试人员连续挂满 14 天，中途账号还被封过一次，重新走了一遍流程。这三件事我都写过。"
cover:
    image: "/images/gplay-personal/cover.png"
    alt: "交完 25 美元才知道，Google Play 给自己装应用不要钱"
    relative: false
---

十月我为了一个开发者账号忙了小半个月。25 美元报名费，身份验证要政府签发的证件，付款资料的地址必须和证件上一字不差，新注册的个人号还得先找 12 个测试人员连续挂满 14 天，中途账号还被封过一次，重新走了一遍流程。这三件事我都写过。

就在我跑这些流程的时候，10 月 13 日，谷歌在 Play 的产品博客上发了一篇更新，标题是 New on Google Play: More ways to create, discover, and play。

里面有这么一句。

> Using Google AI Studio, you can vibe code a custom Android app. Since many of these apps start out as personal projects, through Google Play you can now launch these personal apps on your own device. This means your creations stay private, secure, and separate from the public store catalog. You only need your standard Google account to do so — no developer account required.

用大白话说，对着 AI 说几句话生成一个安卓应用，点一下发布，扫个二维码，应用就装到你手机上了。不用开发者账号，不用 25 美元，不用审核。

同一个 Google Play，两套门槛。

![开发者账号和个人库，同一个 Google Play 上的两套门槛](/images/gplay-personal/gplay-two-doors.png)

## 这条路具体怎么走

Play Console 的帮助中心新开了一页，标题叫 Personal library apps in Google Play，把步骤写得很细。

在电脑浏览器打开 Google AI Studio，用你的个人 Google 账号登录。点 New app，选 Build an Android app，输入你想做什么。生成预览之后，右上角点 Publish。第一次用要连一下 Google 账号，然后点 Create app。

几分钟后，屏幕上出现一个二维码。

手机用同一个 Google 账号登录过 Play 商店，扫这个码，装。

不想扫码也行，路径是 Play 商店应用里点个人资料，管理应用和设备，找到 Your personal library，里面就是你自己做的应用，点安装。更新也一样，回 AI Studio 改 prompt，点 Update app，手机上自动更新。

整个过程我最想确认的是那句 no developer account required。官方帮助页写得更死。

> You only need a Google Account to get started.

一句话，一个账号，一个二维码。

![从一句 prompt 到装进手机，官方帮助页写的五步](/images/gplay-personal/gplay-flow.png)

## 有开发者账号的人反而用不了

这是我读到最意外的一条，官方帮助页单列了一个问答。

> Owners of Google Play Console developer accounts will need to use a different Google Account to use this feature.

意思是，如果你已经有 Play Console 开发者账号，想用这个功能，得换一个 Google 账号。

谷歌没解释原因。我的理解是账户体系要隔离，个人库里的东西不能和你在 Play Console 里以开发者身份提交的应用混在一起，否则你用一个号既能提交公开应用又能绕过审核装私人应用，边界就糊了。这是我的推断，官方没写。

但对开发者来说这个体验挺怪的。你为了正经发布交过 25 美元，现在想顺手给自己做个小工具，反而要再注册一个账号。

## 「私人」这两个字要打个引号

官方反复强调私人这两个字。不进公开目录，不出现在搜索结果、榜单和推荐里，只能你自己装。

但它不等于谷歌看不见。

帮助页末尾那段写得很清楚，当你在 AI Studio 里创建、修改或发布应用时，工具会把这些东西传给 Google Play：应用的包、创建过程的元数据（包括 prompt）、AI 生成标记、软件依赖和 SDK、用户可见的字符串，还有源代码。

拿去干什么？官方列了四项：安全扫描、滥用防护、政策合规，以及训练和改进自动化的应用安全分类器。

所以你写的每一句 prompt，AI 生成的每一行 Kotlin，都在谷歌的扫描管道里过了一遍。免的是人工审核和上架流程，不是数据采集。

这条我建议大家读原文，因为「私人」在中文语境里容易被读成「没人管」。

## 审核是免了，但不是没规矩

同一页列了明确的内容禁区，个人应用一样不能碰。

不能做银行、贷款、加密货币、真钱赌博、政府服务这几类高风险业务。不能申请无障碍服务、短信和通话记录、绑定通知、请求安装应用这类敏感权限。不能侵犯第三方知识产权，不能冒充已有品牌、政府机构或个人。不能含恶意软件、间谍软件、钓鱼、未授权的数据采集，也不能有仇恨言论、色情内容和暴力。

违规的后果是应用从 AI Studio 里被移除，屡次违规会终止你对 AI Studio 的访问权限。

免审只是省掉了排队和人工核对，该守的线一条没少。区别在于以前是谷歌在上架前拦，现在是你自己撞上去才处理。

## 它生成的是个什么工程

这部分我对写代码的人有用。

AI Studio 生成的是一套标准 Gradle 项目，构建脚本用 build.gradle.kts 的 Kotlin DSL，界面层是 Jetpack Compose 配 Material 3，架构是单 Activity 加 ViewModel 和数据类，还带 AndroidManifest、drawable、strings 这些资源文件。

开了 Firebase 的话会多出 google-services.json、firebase_applet_config.xml 和 firestore.rules。依赖由 agent 自己从 Maven 和谷歌仓库里加。

你可以下载成 ZIP，导进 Android Studio 接着改。

限制也写得很直白，四条。

只有客户端，没有服务端组件，涉及密钥管理、多人协作、Google Workspace 接口的功能都做不了。单 Activity 单 module，结构超过这个规模就不行。只认 Kotlin 和 Jetpack Compose，Java 和 XML 布局不支持。没有 NDK，C 和 C++ 进不来。可穿戴和 Android TV 也不支持，只有手机和平板。

还有个细节挺有意思，浏览器里的模拟器是跑在云端的，你不需要装 Android SDK、Android Studio 或者本地模拟器。但它不支持相机、NFC、蓝牙，GPS 是模拟的，而且没有 Google Play services，谷歌登录和地图在模拟器里用不了，得装到真机才正常。

这套工程的上限很清楚，适合做个人工具、小应用、原型，不适合往上叠大工程。

![AI Studio 生成的安卓工程结构，以及官方列出的四条硬限制](/images/gplay-personal/gplay-limits.png)

## 同一个按钮，两条路

写到这里我发现最值得讲的是 AI Studio 里那个 Publish 按钮的解释。

它有两条出口。一条通向个人库，零门槛，只有你能装。另一条通向 Play Console 的内测轨道，需要 25 美元的开发者账号和完整开发者资料，最多能给 100 个测试人员装。

同一个界面，同一次生成，两套规则差别这么大。

谷歌在博客里把后续路径也铺好了。未来几个月，你可以把个人应用分享给一小圈朋友和家人，那时候应用要先过一遍 Trust & Safety 审核。如果你想让全世界都能用，可以升级成完整开发者账号，个人库里的所有应用会迁移到 Play Console，变成内测轨道上的草稿应用。

我读这段的感觉是，谷歌在把「做个小工具自己用」和「做一个产品给人用」拆成了两个完全不同的入口，中间留了一条升级通道。

以前只有后者，你想给自己手机装个 AI 刚生成的东西，得先变成开发者。现在前者单独开了门，代价是你得先承认自己不是开发者。

![谷歌铺好的三级台阶：个人库、分享给朋友家人、升级开发者账号](/images/gplay-personal/gplay-stairs.png)

## 这功能现在中国用不了

这个功能现在是 beta，只对美国地区的 18 岁以上用户开放，未来几周内逐步铺开。Play 商店应用要 53.5 以上版本。其他国家「near future」，官方没给时间表。

我这边不在支持的地区，功能没摸到。上面这些流程、限制、原文，全部来自三份官方文档：10 月 13 日的 Play 产品博客、Play Console 帮助页的 Personal library apps in Google Play、以及 AI Studio 的安卓构建文档。哪句话是官方原文，哪句是我的推测，正文里都标了。

那问题来了。

如果以后你在美国的朋友对着 AI 说几句话就把应用装上了，而你还在为 12 个测试人员连挂 14 天发愁，你会去注册一个普通 Google 账号绕过去吗？还是说，这 25 美元和那五道关，本来买的就是「能给别人装」这个资格？

再想一层。谷歌这次拆的其实只是「给自己用」这半堵墙，另一半一动不动。等到哪天 vibe coding 出来的人都想分享给朋友了，这道 Trust & Safety 审核会不会又变成新的五道关？

你手上有没有那种「就我自己用，但一直懒得为它开账号」的小工具？如果现在只要一句 prompt 就能装上手机，你会先做一个什么？
