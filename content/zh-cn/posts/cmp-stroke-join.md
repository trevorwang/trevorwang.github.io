---
title: "你写的圆角，CMP 桌面端画成了尖角"
date: 2026-11-12T08:40:00+08:00
draft: false
categories: ["公众号"]
tags: ['Kotlin', 'Compose Multiplatform', 'Skia']
summary: "Stroke(join = Round) 在 CMP 桌面端、iOS、Web 上全被画成 Miter，Android 一直没事。根因是一个从不执行的 setter。两个版本各渲一套图逐像素对比，差 1,619 个像素。"
cover:
    image: "/images/cmp-stroke-join/cover.png"
    alt: "你写的圆角，CMP 桌面端画成了尖角"
    relative: false
---

写 `Stroke(join = StrokeJoin.Round)`，画出来是尖的。这个 bug 在 Compose Multiplatform 的桌面端、iOS、Web 上都成立，Android 一直没事。

1.13.0-beta02 把它修了。发布日期 2026 年 11 月 4 日，一条 release note 就一句话：让 `Stroke(join = Round)` 和 `Stroke(miter = 0f)` 不再被忽略。

我在本机照着验了一遍，两个版本各渲一套图，逐像素对比。

## 先说怎么复现

不用起窗口。Compose 桌面端有个 `ImageComposeScene`，给它宽高和密度，把 Compose 树渲成 `Image`，再编码成 PNG。

一个 360×300 的画布，密度给 2，出来是 720×600 的图，够看清楚拐角。

我也就两个值在变：依赖版本，和 `Stroke` 的参数。

![工程与出图方式](/images/cmp-stroke-join/fig1.png)

## 同一份代码，两种形状

上面这一组是 `join = Round`，宽度 30，端点也给了圆头。

![join 与 miter 在两版下的四种渲染](/images/cmp-stroke-join/fig2.png)

左边三条拐角全是尖的，顶上还带着小刺。右边是圆的，尖刺没了。

下面一组是 `miter = 0f`，我特意画了个锐角的 V 形。左边保住了长长的尖角，右边被切平了。

按 Skia 的规矩，miter limit 给 0 就等于不要尖角，所有接头退化成斜切。1.12.2 里这个 0 是白写的。

## 数字比图更硬

我把同版本内的两张图互相做差，再把两版对应的图做差。

![差异像素的位置和数量](/images/cmp-stroke-join/fig3.png)

在 1.12.2 里，`join=Round` 和 `join=Miter` 两张图逐像素完全一样，PNG 字节数也一样，都是 8,905 字节，哈希一个字符不差。

`miter=0f` 和 `miter=4f` 同样完全一样，都是 4,176 字节。

换成 1.13.0-beta02，同样是这两组对比，一个差 1,619 个像素，一个差 414 个。整张图 108,000 个像素，前者占 1.5%。

差异全在三个拐角的尖端上，就是上图品红色那几块。

## 根因是一个不执行的 setter

Compose 的 `Paint` 是个包装，里面装着 Skia 的 `Paint`。属性写成这样：

![修之前与修之后的默认值](/images/cmp-stroke-join/fig4.png)

`strokeJoin` 有自定义 setter，赋值的时候会把 Skia 那份一起改掉。但属性初始化不走 setter，只填了 Kotlin 的字段。

于是包装层说默认是 Round，Skia 那边实际是 Miter，两边分叉了。

真正致命的是后面这一步。绘制的时候，赋值之前有个判断：值跟当前字段一样就跳过。你写 `join = Round`，字段里本来就是 Round，判断为真，setter 根本没被调用，Skia 从头到尾不知道你改过。

`miter = 0f` 同理，字段默认就是 0f。

顺带说一句 `filterQuality`，它连 setter 都没有，直接是个普通字段，值改了也不会通知 Skia。这类字段在这套包装里不止一个，只是不影响形状的那些，没人会注意到。

这个 issue 是 Tim Haywood 三个月前在 YouTrack 上开的，CMP-10688，标题直接写「incorrectly renders StrokeJoin.Round as Miter」。他把根因、Skia 源码位置、潜在修法都写清楚了，甚至连「把 Kotlin 字段删掉、直接代理到 Skia」这个方向都提了。

修的人是 Ivan Matkov，PR 是 compose-multiplatform-core 的 #3528，10 月 26 日合并进 release/1.13 分支，前后审了两轮。从 issue 到合并，中间三个月。

## 修法反而是把默认值改对

PR 的改动小得让人意外：把三个默认值改掉，让它和 Android 对齐。

`strokeJoin` 从 Round 改成 Miter，`strokeMiterLimit` 从 0f 改成 4f，`filterQuality` 从 Medium 改成 Low。

改完，包装层的字段和 Skia 的实际状态一致了。你写 Round 跟字段里的 Miter 不同，判断为假，setter 正常执行，参数就传下去了。

所以这不算修绘制代码，是把谎报的账目对齐。

## 对照组的 0 说明没改坏别的

我最担心的是这种底层改动顺手把别的地方带歪。所以留了对照组：`join = Miter` 的图，两版互比。

0 个像素差。一个字节都不差。

1.13.0-beta02 里那一列圆角和斜切的改动，只落在受影响的路径上。

## 为什么这么久了才被看见

我猜三个原因叠在一起。

一是默认的 `join` 就是 Miter。多数图表、图标、进度条根本不传这个参数，形状对了，就没人往下查。

二是显式传 `Round` 的地方，画出来只是「不够圆」，不像崩溃那样刺眼。设计师提一句「这个角好像有点尖」，工程师十有八九会怀疑是自己路径画错了。

三是它只在非 Android 上发生。同一份代码在真机上是对的，在桌面预览里是尖的，人很容易归因到预览工具的渲染误差，而不是认定框架有 bug。

## 只有非 Android 受影响

diff 落在 `compose/ui/ui-skiko/src/nonAndroidMain/`。Android 走的是 `android.graphics.Paint`，那边的默认值本来就是 Miter、4f、Low。

所以这个 bug 有一个很讨厌的性质：你在 Android 真机上看不出问题，一跑桌面端或者 iOS 就变形了。写跨平台 UI 的人在自家机器上看到的，和别人看到的不一定是同一个形状。

## 第三个默认值我量不出差异

`filterQuality` 从 Medium 改成 Low，理论上影响图片缩放的采样质量。我拿一张 256 像素的棋盘图，分别缩到 40 和 80，又拿 8 像素放大到 320。

三次对比，全是 0 个像素差。

原因是 `drawImage` 这个 API 自己带 `filterQuality` 参数，默认值不经过 `Paint`。只有你自己造 `Paint` 去画的时候，这个改动才有意义。所以它是隐患，不是现象。

查了一圈没找到能观测的路径，我就照实写了，不给它编一个场景。

![三个默认值改了什么](/images/cmp-stroke-join/fig5.png)

## 你现在该做什么

项目还在 1.12 或者更早，并且用了 `join = Round`、`miter = 0f` 这类看起来没生效的参数，那就是中招了。

两条路。要么升到 1.13.0-beta02，接受它还在预发布。要么先把形状改成不依赖这两个参数的画法，比如自己描一条圆角折线。

想确认自己有没有中招，不用读代码。写一个 `ImageComposeScene`，同一个形状渲两遍，一遍 `join = Round` 一遍 `join = Miter`，两张 PNG 做差。差 0 个像素就是中招，差几千个才是正常的。这个方法对我这套图很灵，对任何「参数看起来没生效」的绘制问题也应该灵。

升之前有件事得知道：1.13 里 `Paint()` 加了个前置条件，直接 new 会抛 `No Compose UI graphics implementation is registered`。要么先建一个 scene，要么走正常的绘制流程。我第一次跑就撞在这，改成先渲一张图热个身才好。

至于这个 bug 在线上躺了多久，我不知道，issue 是三个月前开的，代码可能更早。这种「参数写对了但没生效」的坑，只能靠像素级的对比才看得见。
