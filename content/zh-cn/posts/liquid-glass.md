---
title: "Compose 桌面跑通了 Liquid Glass"
date: 2026-11-06T08:05:00+08:00
draft: false
categories: ["公众号"]
tags: ['Compose Multiplatform', 'KMP', 'Liquid Glass']
summary: "苹果的 Liquid Glass，安卓一直没有官方答案。我把 Kyant0/Backdrop 装进一个 Compose Multiplatform 桌面工程，在 macOS 上真的跑通了：跨端适配只写了 280 行，因为 AGSL 是 SkSL 的子集。"
cover:
    image: "/images/liquid-glass/cover.png"
    alt: "Compose 桌面跑通了 Liquid Glass"
    relative: false
---

去年苹果在 iOS 26 上放出 Liquid Glass 的时候，安卓这边最热闹的讨论只有一句：我们什么时候能有。

官方的答案是没有。Jetpack Compose 到今天也没有内置这种材质，你想在界面上放一块会折射、会随背景变色的玻璃，要么自己写着色器，要么找一个库。

AndroidLiquidGlass 就是这样一个库，作者 Kyant0，4,509 星。名字里带 Android，但它在描述里的定位是 Compose Multiplatform，主分支直接叫 kmp。

我一直好奇一件事：这种重度依赖平台图形能力的视觉效果，跨端到底能还原几成。

安卓上画玻璃靠的是 AGSL 着色器，桌面靠 Skia。两条完全不同的图形栈，一个库想同时喂饱它们，通常的结局是凑合能用，效果打折。

所以我做了件笨事：在本机建了个最小的 Compose Multiplatform 桌面工程，把库加进去，在 macOS 上让玻璃真的画出来，然后一个效果一个效果拆开看。

结论放前面：能跑，效果也不是凑合。整库 2,550 行，跨端适配只写了 280 行，原因在源码里，后面会说到。

## 先认清它是谁

仓库叫 `Kyant0/AndroidLiquidGlass`，4,509 星、364 个 fork，Apache-2.0。名字里带 Android，但描述写的是 Compose Multiplatform Liquid Glass effect，主分支叫 kmp。

库现在的正式名字是 Backdrop，Maven 坐标 `io.github.kyant0:backdrop`，最新版 2.0.1，八月二十六号发的。

README 里有句话值得抄下来：这个库不包含任何高层组件，组件要你自己搭。它给的是效果原语，不是一行调用就出来的现成按钮。

体量也小：桌面端 jar 136 KB，连它依赖的 shapes 库 46 KB 一起，总共不到两百 KB。

![](/images/liquid-glass/fig1.png)

## 最小工程就三步

新建一个 Gradle 工程，装 Kotlin 2.4.10、Compose Multiplatform 1.12.0 两个插件，然后加依赖：

`implementation("io.github.kyant0:backdrop:2.0.1")`

代码就一个模式，记住「谁被采样」和「谁去采样」必须配对：

后台的内容用 `Modifier.layerBackdrop(backdrop)` 注册成采样源，前面的面板用 `Modifier.drawBackdrop(backdrop, shape, effects)` 去取。

我用一个画了四个色块加一层细网格的 Canvas 当背景，前面放一个圆角二十六的面板，效果里依次调 vibrancy、blur、lens。窗口一开，玻璃就出来了。

![](/images/liquid-glass/fig2.png)

## 它长什么样

我把四种效果拆开各跑一格，同一张背景，同一个面板，只改效果栈。

第一格只给形状，什么都不加，你会发现面板里透出的网格是清晰的。这一点很关键：这说明它不是预先烘焙一张模糊图，而是每帧真的去采样后面的内容。

第二格加 8dp 模糊，网格糊成一片。第三格叠 vibrancy，颜色明显更艳。第四格再加 lens，面板边缘开始出现那圈像水滴一样的折射。

![](/images/liquid-glass/fig3.png)

vibrancy 到底是什么，官方文档写得很实在：就是把饱和度乘 1.5，等价于 `colorControls(saturation = 1.5f)`。

## 它其实给了五种 backdrop

文档里 Backdrops 那一页列了五种，用途分得很细。

`rememberLayerBackdrop` 最常用，把某个 Composable 的内容录成采样源，必须配 `layerBackdrop`。

`rememberCombinedBackdrop` 把多个 backdrop 合成一个，用来做标签栏和滑块这种跨区域采样。`rememberCanvasBackdrop` 不依赖布局坐标，可以往一块空 backdrop 上画自己的东西。`emptyBackdrop` 就是什么都不画。

还有一个 `rememberBackdrop`，让你用自定义命令直接画。

分这么细的原因是不这么分就做不出高级效果。比如底部标签栏的玻璃要能采到「整页内容」，而不是只采自己后面那一小块，那就得先合成。

这部分是我觉得它比同类库扎实的地方：不是给一个 API 让你套，是把采样这件事的各种边界都拆出来给你。

## 一个必须自己补的坑

我第一版在浅色背景上跑，面板几乎是白的，字全看不清。

原因不怪库：玻璃本来就应该透，浅色背景上它自然也是浅色，白字自然就没了。iOS 上系统会替你压一层材质，这库不管这层。

解法是自己加。`drawBackdrop` 有个 `onDrawSurface` 参数，我在里面画了一层 30% 的黑：

`onDrawSurface = { { drawRect(Color(0x30000000)) } }`

加完两边都正常了，浅色深色各一张对比图。

![](/images/liquid-glass/fig4.png)

## 为什么桌面能跑

这是我最想弄清的一点。库名叫 Android，实现里也真的用了 `android.graphics.RuntimeShader`，那桌面凭什么能画？

翻源码就明白了。整个库分三块：commonMain 2,270 行写所有逻辑，androidMain 146 行、skikoMain 134 行负责两端落地。

着色器是 AGSL 字符串。安卓侧把它塞进系统的 RuntimeShader；Skia 侧直接喂给 `RuntimeEffect.makeForShader`，因为 AGSL 本来就是 SkSL 的子集。

也就是说 Android 上那套玻璃着色器，原样搬到 Skia 上就能编译。

桌面、iOS、wasm 走的都是这条 skiko 路径。

两端的支持判断也写在源码里：安卓要 `SDK_INT >= 31` 才有 RenderEffect，要 `>= 33` 才有 RuntimeShader；Skia 那边两个函数直接返回 true，不设门槛。

## 还有两个不那么顺的地方

第一个在文档里，但容易被忽略：lens 效果要求 `shape` 必须是 `CornerBasedShape`。我用 `RoundedCornerShape` 所以没事，换成自定义 Shape 就得自己处理。

第二个在源码里，文档没提：Skia 实现不支持整型数组 uniform。`setIntUniform(name, values: IntArray)` 在桌面端会直接抛 `UnsupportedOperationException`，安卓侧是好的。这类差异平时碰不到，一旦你写自定义着色器就会撞上。

顺带说一句文档的标记方式。官方 API 页写着「效果本质上是 RenderEffect，Android 12 起生效，涉及 RuntimeShader 的要 Android 13」。这句在桌面语境下不成立，我在 macOS 上用 lens 一点问题没有。它那句是站在安卓角度说的，别当成平台限制。

## 性能和我踩的边界

我这台 M 系列 Mac 上，两块玻璃一起动，帧率稳定在 60，正好顶到刷新率。加到九块玻璃同屏、每块都带动态 lens，掉到 58.7 帧。

也就是说这个效果的开销，在这个量级上还没成为瓶颈。

编译方面，第一次带依赖解析跑了 25 秒，之后靠 Gradle 缓存基本一秒内。

这里也把截图方式说清楚：macOS 没给屏幕录制权限，系统级截屏只能拍到壁纸。所以我改用 SkiaLayer 自带的 `screenshot()`，直接拿窗口画面写 PNG，像素是准的。

![](/images/liquid-glass/fig5.png)

## 社区情况

4,509 星不是白来的，这个效果确实抓人。但节奏摆在那：提交停在八月二十六号，文档站的更新时间写着「5 个月前」。

40 个 open issue，官方 FAQ 里第一条是「为什么效果不生效」，答案是大概率忘了加 `layerBackdrop`。第二条更直接，是一个安卓端的 SIGSEGV 崩溃，在 RenderThread 里炸，要照着教程去改。

这库是好东西，但它是个人项目，不是官方组件。指望它像 Compose Foundation 那样稳定，会失望。

## 我的判断

想在 Compose 桌面上做点 macOS 味的玻璃面板，这个库现在就能用，成本两百 KB 加二十行代码。

但别把它当成品组件用。它给的是效果，交互、可访问性、深色浅色的适配全是你自己的活。苹果那套东西好看，一半功劳在系统替你处理了这些边角。

我打算把它用在自己的小工具上试试。真正想验证的是那件事：一个 Android 原生库，靠 AGSL 和 SkSL 的这层血缘，在桌面上跑得跟原生一样。

你的跨平台项目里，会为了一个视觉效果去引一个个人维护的库吗？
