---
title: "用 KMP 做一个自己的 PikPak 客户端：expect/actual 一次都没用上"
date: 2026-09-27T08:30:33+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin"]
summary: "前两天刷到一个小众软件的帖子，一个叫 Piko 的项目，开发者自荐：开源的 PikPak 第三方客户端，Android 和 Windows 双端，两端共用界面与业务代码。"
cover:
    image: "/images/kmp-piko/cover.png"
    alt: "用 KMP 做一个自己的 PikPak 客户端：expect/actual 一次都没用上"
    relative: false
---

前两天刷到一个小众软件的帖子，一个叫 Piko 的项目，开发者自荐：开源的 PikPak 第三方客户端，Android 和 Windows 双端，两端共用界面与业务代码。

网盘客户端这东西不算新鲜，但「一套代码双端跑」这句话我今年听得太多了，多到有点麻木。所以真正让我停下来的不是它支持双端，是它把双端这件事做成了什么形状。

我把仓库拉下来数了一遍，然后在一个地方愣住了。

## 先把结构摆出来

Piko 是 Kotlin Multiplatform 加 Compose Multiplatform，五个模块，实测的代码量是这样：

- `shared` 状态与业务，`commonMain` 79 个文件
- `ui` 共享界面，`commonMain` 74 个文件，**只有这一个 source set**
- `app` Android 入口，22 个文件
- `desktopApp` 桌面入口，29 个文件
- `cli` 开发用的命令行工具，不上线

![Piko 模块与 source set 分布](/images/kmp-piko/01-modules.png)

前两个加起来 153 个文件、3 万 1 千多行 Kotlin，是两端共用的部分。后两个是两端的入口，加起来 51 个文件、8 千多行。

看到这里还算正常。不正常的是我接着敲的这条命令。

## 一行 expect/actual 都没有

在 KMP 里，平台差异的教科书答案就是 expect/actual：公共代码里写 `expect fun`，各平台 source set 里写 `actual fun`。我本来想数数它用了多少处，好写进文章当例子。

```
grep -rn "^\s*expect " --include="*.kt" shared ui app desktopApp
expect 声明: 0
actual 实现: 0
```

我以为是自己的正则写错了，换了几种写法，结果一样。一个 3 万 1 千行共享代码、全仓库 4 万 6 千行 Kotlin 的项目，expect/actual 用了零次。

更奇怪的是 source set 的分布。它的 `shared` 里没有 `androidMain`，`ui` 里连 `desktopMain` 都没有，`shared` 唯一一个平台相关的 source set 叫 `jvmSharedMain`，里面只有一个文件、67 行。

这就不是「用得少」了，是压根换了一套做法。

## 它的答案：接口，加运行时注入

作者在自己的架构笔记里写了一句话，我觉得比教程里任何一段都值钱：

> 平台差异用接口，不用 expect/actual

理由是这句注释，写在能力的接口定义上：

> 实现要拿 Context、窗口与各自的播放后端，这些都是运行时对象，不是编译期的平台差异。

这句话点破了我以前的糊涂账。expect/actual 解决的是「编译期差异」，同一个函数名，两个平台各写一份实现，编译器帮你挑。但客户端真正要分家的东西，几乎全是运行时对象：Android 要拿 Context、要注册 ActivityResult 启动器、播放要走 libmpv 的 SurfaceView；桌面端要窗口句柄、要 MPV 的原生 DLL、播放画面要零拷贝进 Skia。这些东西的共同点是，它们在编译期看起来是同一种东西，只是构造方式不同。

而构造方式不同，恰恰是依赖注入最擅长的事。

它是这么做的。共用的 `ui` 模块里定义一个接口，叫 `PikoPlatform`，把界面要跟平台要的能力全列在里面：剪贴板、系统取色、下载目录选择、本地文件的打开与分享、全屏对话框、应用内更新。

两个入口各实现一份，`AndroidPikoPlatform` 和 `DesktopPikoPlatform`，最后在入口处拼进根界面 `PikoApp(services, platform, appearance, videoPlayer)`。界面内部通过 `LocalPikoPlatform` 取用，不再回头找单例。

## 平台没有的能力，用 null 说出来

接口方案最容易被嫌的地方是啰嗦：同样是「分享文件」，Android 有系统分享面板，桌面端没有对应物，那接口该怎么写。

它的处理是这样的。能力字段可以是可空的：

```kotlin
/** 为 null 表示该平台没有应用内更新。 */
val updater: AppUpdateService?

/** 为 null 表示该平台不提供片段下载的画面预览，入口随之隐藏。 */
val videoPreview: VideoPreviewSupport?
```

也有的写成布尔：

```kotlin
/** 系统分享面板。桌面端没有对应物时为 false，界面不显示「分享」。 */
val canShare: Boolean
```

还有更细的：软键盘是否弹出，桌面端恒为 false；快捷键主修饰键，Windows 是 Ctrl，macOS 给的是 ⌘，由一个 `shortcutModifier` 属性吐出来。

这套约定带来的好处是，界面只需要问一次「你能不能」，然后自己决定藏不藏入口，不需要为了一个只有单端存在的功能，在公共代码里到处塞平台判断。这类判断一旦多起来，公共代码就名存实亡了。

## 第三种手段：中间 source set

接口解决的是「两端真的不一样」。但还有一类代码是「两端完全一样，只是要用 JDK 的 API」。Piko 里这类代码的样本是网络代理选择器 `PikoProxySelector`，67 行。

它要把一个进程级的 `ProxySelector` 装进 JVM，理由是 SDK 内部自己建 OkHttpClient，拿不到构造参数，只能从全局进去。这段代码在 Android 和桌面端一字不差。

如果按教科书走，这就是两份 `actual`，内容复制粘贴。它的做法是在中间加一层 source set，用构建脚本声明这个组里同时装两种 JVM 目标：

```kotlin
applyDefaultHierarchyTemplate {
    common {
        group("jvmShared") {
            withCompilations {
                it.platformType == KotlinPlatformType.jvm ||
                    it.platformType == KotlinPlatformType.androidJvm
            }
        }
    }
}
```

`jvmSharedMain` 这个目录就这么来的。写完一份，两个 JVM 系目标都编译它。这个手段我在这篇文章之前没在任何教程里见过，但它是三个手段里省事最多的一个。

所以到这儿，KMP 划边界的手段其实有三档：

1. 能共用就直接放 `commonMain`，这是绝大多数代码的位置
2. 两端行为真的不同，用接口加运行时注入
3. 两端代码相同、只是要用 JDK API，加一个中间 source set

expect/actual 是第四档。它没被用，是因为前三档刚好覆盖了这个项目的全部需求。我现在的看法是，expect/actual 更像是给「纯声明式差异」准备的，比如一个常量、一个没有运行时状态的函数。真正复杂的客户端，落到那一档的东西比想象中少。

![划平台边界的手段分层](/images/kmp-piko/02-layers.png)

## 那 3 万 1 千行共享代码长什么样

`ui/src/commonMain` 是整个界面：主题、导航、网盘、传输、设置、回收站、登录，加上各种组件。入口是 `PikoApp`，两端各自把依赖拼好再传进去。

Android 这边在 `Application` 里装配：

```kotlin
val clientManager = PikoClientManager(AndroidPikoSessionStore(this, sessionManager), appScope)
services = PikoServices(
    preferences = sessionManager,
    clientManager = clientManager,
    downloadManager = PikoDownloadCoordinator(
        storage = AndroidPikoDownloadStorage(this, sessionManager, appScope),
        segmentDownloader = AndroidPikoSegmentDownloader(this),
        // ...
    ),
    uploadSources = AndroidPikoUploadSources(this),
)
platform = AndroidPikoPlatform(this) { appUpdater }
```

桌面端在 `main` 里装配，形状一样，实现换了一套：`DesktopPikoPlatform(settings)`、`DesktopPikoDownloadStorage`、`FilePikoSessionStore`。会话存储这个例子最能说明问题，公共接口只有 21 行：

```kotlin
interface PikoSessionStore : SessionStore {
    suspend fun loadLastAccount(): String?
    suspend fun saveLastAccount(account: String)
    suspend fun loadCredentials(account: String): PikoCredentials?
    // ...
}
```

Android 侧走 DataStore，桌面端往 `~/.piko` 下写文件。界面和业务代码完全不知道这件事。

这里有个我觉得挺值得留意的对照：接口规定了两端要做什么，但管不住两端实现得差多远。我翻了桌面端那份实现，它把会话、账号、密码分开写在 `~/.piko` 的三个文件里，密码那一个是明文写的。自己的机器上自用没什么问题，多人共用的机器上就值得掂量一下。同一件事，同一份接口，两端的严谨程度可以差出一截。

## 踩坑提醒

它的架构笔记里记了不少实测出来的坑，我挑几个跨平台的记下来。

**两端的 Compose 版本可以对不上，同一行代码可能只在一端报错。** 它桌面端的界面库钉在 Compose Multiplatform 1.12.0，Android 端由 BOM 决定。原因很具体：CMP 1.13 带的 skiko 把渲染后端包进了 `OnScreenRedrawer`，而它用的播放库要从里面直接拿 `Direct3DRedrawer` 做 D3D11 画面直通，一开播放器窗口就抛异常。所以这个项目里，升级 CMP 不是改一个数字的事，得等上游播放库跟上。

**Android 端的 Compose 反而要退回稳定版。** 它的版本目录里写着：1.13 的 alpha 在列表快速滚动时崩溃，可交互的 ListItem 在测量阶段读副文本基线，为对齐而摆放的子节点去查找尚未入表的父节点。BOM 因此换成稳定版，ui 与 foundation 落到 1.12.x。

**改共用模块必须两端都编译。** 作者把这句话直接写进了给协作者看的文件里，因为 `shared` 和 `ui` 的改动会同时波及两个入口模块，只编译一端看不出来。而两端的 material3 版本本来就不是一个，同一行代码完全可能只在一端报错。

**加一个偏好项要同时改三处。** 公共接口一处，Android 的 DataStore 实现一处，桌面的设置存储一处。同类的还有全局导航栈：主界面的位置放在仓库层并持久化，所以目录选择器这类浮层必须自己维护一份路径栈，动了仓库层那份，主界面的位置会跟着跑。

**依赖要用 api 而不是 implementation，是有原因的。** `shared` 用了 `api(libs.cmp.runtime)`，因为它对外暴露的状态类型直接是 Compose 的 `State`，消费方拿得到这些类型才能编译。反面例子是 kotlinx-serialization：SDK 那边以 implementation 声明，不会传递过来，调用方得自己引一份。

**一个许可证细节：源码是 MIT，但 Android 安装包整体是 GPLv3。** 因为它内置的 mpv 与 FFmpeg 是以 GPL 构建的。自己用没关系，想二次分发要看清楚这一点。

## 它跑起来是什么样

Piko 的官方 README 里有各端的截图，我挑两张放这儿，上面是 Android，下面是同一个窗口里的桌面端。

两张图值得对比的不是内容，是导航。手机上导航栏在底边，桌面窗口里挪到了左侧，中间那层判断只看窗口宽度，不看是什么设备。所以桌面缩窄到一定程度会变成手机的样子，平板分屏走的是同一套判断。这层自适应也写在共用代码里。

![Piko Android](/images/kmp-piko/phone.jpg)

![Piko Desktop](/images/kmp-piko/desktop.jpg)

功能上它做得比我预期的多：磁力链接能按作品、分区、集数解析，字幕跟着视频归组，广告文件自动排除，保存之前可以先完整预览再勾选要的文件。播放进度跟 PikPak 官方客户端同步，下载和在线播放都是 8 条连接并发。还支持无损截取视频片段、上传、服务端解压、全盘搜索、查找重复、看网盘用量和本月流量。

值得一提的是它的开发方式，提交记录里能看到 Claude 的协作者署名，README 里也写了用 LLM 辅助写代码时的要求：理解新增代码的逻辑，并在真机上验证。作者在帖子里说配色这块自己审美欠缺，这套界面是 M3 生成加人工调的。

## 最后

小项目要双端，我的建议是先把 `shared` 和 `ui` 切出来，让它俩干干净净地只剩 `commonMain`，平台实现全部塞进两个入口模块。这样每次想偷懒往共用模块里加平台判断的时候，都会先卡一下。

`PikoProxySelector` 那 67 行给我印象最深。它本来最像需要 expect/actual 的地方，最后是加了一层 source set 解决的。手段够多的时候，你反而不会去动那个最笨的。

项目地址：https://github.com/NihilDigit/piko
