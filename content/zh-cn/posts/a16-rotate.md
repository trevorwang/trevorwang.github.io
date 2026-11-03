---
title: "Android 16 删了你的竖屏锁，我在模拟器上验了三遍"
date: 2026-11-03T09:30:00+08:00
draft: false
categories: ["公众号"]
tags: ['Android', '大屏适配', 'WindowSizeClass']
summary: "同一份代码编出 targetSdk 35 和 36 两个包，把屏幕做成平板再命令它旋转。35 被锁死还有黑边，36 直接横过来；换回手机尺寸，限制又回来了。"
cover:
    image: "/images/a16-rotate/cover.png"
    alt: "Android 16 删了你的竖屏锁，我在模拟器上验了三遍"
    relative: false
---

这两天中文圈在传「谷歌下场施压，逼开发者适配折叠屏」。

我去翻了 [Android 16 的行为变更文档](https://developer.android.com/about/versions/16/behavior-changes-16)，官方用的词不是「鼓励」，是 ignore。忽略你的竖屏锁，忽略你的不可调整大小，忽略你的宽高比上限。

它不是劝你，是直接把这个开关从系统层面拿掉了。

我做了一组对照实测：同一份代码，编译出 targetSdk 35 和 36 两个包，装进同一个模拟器，把屏幕做成平板大小，然后命令它旋转。

## 官方原话是这样写的

> For apps targeting Android 16 (API level 36), orientation, resizability, and aspect ratio restrictions no longer apply on displays with smallest width >= 600dp.

条件是两条：目标版本 36，且屏幕最小宽度到 600dp。满足就生效，应用铺满整个窗口，连黑边都不给。

被忽略的东西列得很具体。清单属性三个：`screenOrientation`、`resizeableActivity`、`minAspectRatio`、`maxAspectRatio`。运行时方法两个：`setRequestedOrientation()` 和 `getRequestedOrientation()`。

竖屏、横屏、反向竖屏、传感器竖屏、用户竖屏，这一串取值全部失效。

![Android 16 到底忽略了什么，例外在哪，逃生门能开到什么时候](/images/a16-rotate/fig1.png)

## 官方为什么敢直接拿掉

Android Developers 文档写道，现在的应用要跑在手机、平板、折叠屏、桌面模式、车机、电视上，还要跑在分屏和桌面窗口这类多窗口形态里。文档的原话是，锁方向、锁大小这类做法在今天「过于限制性」。

说白了，一个竖屏锁在手机上保护了你的排版，在平板和桌面窗口上就变成了让用户没法用。

官方也点了名，说要改成「应用预期自己适配各种方向、尺寸和宽高比」。这话听着软，落到系统行为上是硬的。

## 我的实测怎么搭的

一个 Activity，纯 Java，不引 Compose，界面上只打印四行字：当前 orientation、screenWidthDp、screenHeightDp、smallestScreenWidthDp。

清单里故意把限制全上满：`android:screenOrientation="portrait"`、`android:resizeableActivity="false"`、`android:maxAspectRatio="1.5"`。

同一份源码编两遍，只改 targetSdk，两个包各 2.5 MB。设备是 API 36 的模拟器。

把屏幕做成平板：`wm size 1200x1920`、`wm density 240`，算下来应用区 1280×800dp，最小宽度 800dp，刚好过线。然后用 `settings put system user_rotation 1` 命令它横过来。

## 结果一：targetSdk 35 被锁死，还挨了一句提醒

35 那个包老老实实保持竖屏，内容还是 800×1200，被塞进一块 1280×800 的横屏窗口里，左右两条大黑边。

这是老行为，叫兼容模式。系统还在右下角弹了一句：Tap to restart this app for a better view。

翻译过来是「点一下重开，看得清楚点」。系统的态度已经写在脸上了。

## 结果二：targetSdk 36 直接违背我的清单

换成 36 那个包，同样一条旋转命令，它横过来了。

界面上自己报：orientation 等于 LANDSCAPE，screenWidthDp 1280，screenHeightDp 800，smallestScreenWidthDp 800。铺满，没有黑边。

清单里那三行限制，一行都没起作用。

![同一份代码，两个目标版本，同一块屏幕](/images/a16-rotate/fig2.png)

## 结果三：换回手机尺寸，限制又回来了

我又把屏幕改回手机（1080×2400，420dpi，最小宽度 411dp），再让 36 那个包旋转。

它没动，还是 PORTRAIT。

所以这条规则不是「新版就一定不管」，是「最小宽度过 600dp 才不管」。411dp 的手机上，你的竖屏锁照样有效。

这一点很关键，因为很多转载把它写成了「Android 16 取消了方向限制」，那是错的。

![三组对照：阈值就是 600dp](/images/a16-rotate/fig3.png)

## 例外，还有一道临时逃生门

据 Android 16 行为变更页原文，例外有三个：游戏（按 `android:appCategory` 判断），用户在系统设置的宽高比里主动选了应用默认行为，以及屏幕小于 600dp。

还有一个 opt-out，写在清单里：

```
<property android:name="android.window.PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY"
          android:value="true" />
```

可以写在 activity 上，也可以写在 application 上。写了就退回兼容模式。

但它有保质期。同一份文档提到，这条 opt-out 在未来的 Android 版本里、以 API 37 为目标时不再生效。也就是说，这是给你一年时间搬家，不是给你一个永久开关。

想在 36 上提前看效果，不用改 targetSdk，打开兼容性框架里的 `UNIVERSAL_RESIZABLE_BY_DEFAULT` 标志就行。我这次是直接编了两个包，更干净。

## 这个改动会撞出什么

官方文档在 Common breaking changes 一节列了典型症状，我觉得写得比任何二手稿都准：拉伸变形的布局、跑到屏幕外的动画和组件。

这两个现象背后是同一件事：你的布局假设了某个固定的长宽比。写死高度的 RecyclerView、用屏幕宽算出来的 margin、只给了一种尺寸的图片背景，锁着竖屏时全都看不出来，一横过来就露。

还有一类更隐蔽的：以前锁竖屏，Activity 根本不会因为旋转重建，`onSaveInstanceState` 写得糙也没人发现。现在会重建了，没存好的状态就是用户手里的数据丢了。

## 那到底该怎么改

官方的答案只有一个：按窗口尺寸做布局，不要按设备类型做。

窗口尺寸类现在分了五档，看宽度：小于 600 是 compact，600 到 840 是 medium，840 到 1200 是 expanded，1200 到 1600 是 large，1600 以上是 extra large。

高度单独再分三档：480 和 900 两个断点。

断点表在官方的 [Use window size classes](https://developer.android.com/develop/ui/compose/layouts/adaptive/window-size-classes) 那一页。

那页还给了一个很好用的参照。compact 宽度覆盖 99.96% 的手机竖屏，expanded 覆盖 97.22% 的平板横屏。

文档里有一句我建议你抄下来：窗口尺寸类不由设备屏幕大小决定，它不是用来写 isTablet 逻辑的。同一个应用分屏之后窗口只有一半宽，它就该走 medium 那一套。

![五档宽度、三档高度，按窗口算不按设备算](/images/a16-rotate/fig4.png)

还有一件事容易被忽略：允许旋转意味着 Activity 重建次数变多。以前你锁着竖屏，旋转根本不发生，`onSaveInstanceState` 写得糙也没人发现。现在会发生了，状态没存好就是丢数据。

## 我这组实测的局限，得说清楚

第一，我用的是模拟器改屏幕尺寸，不是真的折叠屏。折叠的悬停态、桌面窗口模式、分屏里的 resize，这些我都拿不到真机行为，只能按文档描述写。

第二，我只测了一个 Activity、一个空的界面，验证的是系统层面的限制有没有生效，不是测某个真实 App 的适配质量。

第三，Android Studio 的折叠屏 AVD 我这次没建，因为要下载新的系统镜像，时间上不划算。你要更真的数据，官方有专门的折叠屏模拟器配置，装一个就能拿到悬停态。

能把话说清楚的部分我说死，拿不到证据的部分我标出来，这是我写所有实测稿的规矩。

## 我的判断

这次改动的分量，比「适配折叠屏」这个说法重得多。

过去十年，竖屏锁是一种说白了的设计妥协：手机屏幕窄，横过来排版就崩，索性锁死。现在系统把它拿掉了，等于把「你的布局能不能自适应」这件事变成了硬指标。

对做产品的人是好事，偷懒的空间少了。对写代码的人是麻烦，`onConfigurationChanged` 和状态保存这两件事得认真写一遍。

你手上那个 App，现在横过来会不会崩？不用等 37，开个模拟器把屏幕调成 1200×1920、density 调成 240，五分钟就知道了。
