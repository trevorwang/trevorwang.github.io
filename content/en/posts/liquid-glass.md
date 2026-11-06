---
title: "Liquid Glass, Running on Compose Desktop"
date: 2026-11-06T08:05:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Compose Multiplatform", "KMP", "Liquid Glass"]
summary: "Apple shipped Liquid Glass and Android still has no official answer. I dropped Kyant0/Backdrop into a Compose Multiplatform desktop project and it actually runs on macOS. The cross-platform adaptation is only 280 lines, because AGSL is a subset of SkSL."
cover:
    image: "/images/liquid-glass/cover.png"
    alt: "Liquid Glass, Running on Compose Desktop"
    relative: false
---

When Apple released Liquid Glass in iOS 26 last year, the liveliest discussion on the Android side came down to one sentence: when do we get it.

The official answer is that we do not. Jetpack Compose still has no built-in material like this. If you want a piece of glass in your UI that refracts and shifts color with whatever is behind it, you either write the shader yourself or you find a library.

AndroidLiquidGlass is one of those libraries. It is by Kyant0 and has 4,509 stars. Android is in the name, but the description positions it as Compose Multiplatform, and the main branch is literally called kmp.

I kept coming back to one question: for a visual effect that leans this hard on platform graphics, how much of it actually survives the trip to another platform?

Glass on Android is drawn with AGSL shaders. On desktop it is Skia. Two completely different graphics stacks, and when a single library tries to feed both, the usual outcome is that it works well enough with the effect dialed down.

So I did the dumb thing. I built a minimal Compose Multiplatform desktop project on this machine, added the library, got glass to actually draw on macOS, then took the effects apart one at a time.

The conclusion first: it runs, and the result is not a compromise. The whole library is 2,550 lines and the cross-platform adaptation is only 280 of them. The reason is in the source, and I will get to it.

## First, what this thing actually is

The repo is `Kyant0/AndroidLiquidGlass`, 4,509 stars, 364 forks, Apache-2.0. Android is in the name, but the description says Compose Multiplatform Liquid Glass effect, and the main branch is kmp.

The library's proper name now is Backdrop, with Maven coordinates `io.github.kyant0:backdrop`, latest version 2.0.1, published on August 26.

One line in the README is worth copying down: the library contains no high-level components, you build those yourself. What it gives you are effect primitives, not a button that falls out of a single call.

It is small, too. The desktop jar is 136 KB, and together with the 46 KB shapes library it depends on, the whole thing is under two hundred KB.

![](/images/liquid-glass/fig1.png)

## Three steps for a minimal project

Create a Gradle project, install the Kotlin 2.4.10 and Compose Multiplatform 1.12.0 plugins, then add the dependency:

`implementation("io.github.kyant0:backdrop:2.0.1")`

The code follows a single pattern. Remember that whoever gets sampled and whoever does the sampling have to be paired:

Background content registers itself as a sampling source with `Modifier.layerBackdrop(backdrop)`, and the panel in front pulls from it with `Modifier.drawBackdrop(backdrop, shape, effects)`.

I used a Canvas with four color blocks and a fine grid as the background, put a panel with a 26dp corner radius in front of it, and stacked vibrancy, blur, and lens in the effects. Open the window and the glass is there.

![](/images/liquid-glass/fig2.png)

## What it looks like

I split the four effects and ran one per cell, same background, same panel, only the effect stack changing.

The first cell gets the shape and nothing else, and you can see the grid through the panel perfectly sharp. That detail matters, because it means the library is not baking a blurred image ahead of time. It samples whatever is behind it, every frame.

The second cell adds 8dp of blur and the grid smears into one mass. The third stacks vibrancy on top and the colors get noticeably more saturated. The fourth adds lens, and that ring of water drop refraction starts to appear at the panel edge.

![](/images/liquid-glass/fig3.png)

As for what vibrancy actually is, the official docs are refreshingly blunt about it. It multiplies saturation by 1.5, which is equivalent to `colorControls(saturation = 1.5f)`.

## It actually gives you five kinds of backdrop

The Backdrops page in the docs lists five, with the use cases split quite finely.

`rememberLayerBackdrop` is the common one. It records the content of a Composable as a sampling source, and it has to be paired with `layerBackdrop`.

`rememberCombinedBackdrop` merges several backdrops into one, for cross-region sampling like tab bars and sliders. `rememberCanvasBackdrop` does not depend on layout coordinates, so you can draw your own content onto an empty backdrop. `emptyBackdrop` draws nothing at all.

There is also `rememberBackdrop`, which lets you draw directly with custom commands.

The reason for splitting them this finely is that you cannot build the advanced effects any other way. Take the glass on a bottom tab bar. It needs to sample the content of the entire page, not just the small patch behind itself, and that requires compositing first.

This is where I think the library is more solid than its peers. It does not hand you one API to wrap around things. It pulls apart every edge of the sampling problem and hands those to you.

## One gap you have to fill yourself

My first version ran on a light background and the panel came out almost white, with the text completely unreadable.

That is not the library's fault. Glass is supposed to be transparent, so on a light background it is naturally light too, and white text naturally disappears. On iOS the system presses a layer of material in for you. This library does not handle that layer.

So you add it yourself. `drawBackdrop` has an `onDrawSurface` parameter, and I drew a 30% black layer inside it:

`onDrawSurface = { { drawRect(Color(0x30000000)) } }`

With that in place, both themes behave. Here is the light and dark comparison.

![](/images/liquid-glass/fig4.png)

## Why it runs on desktop

This is the part I most wanted to understand. The library is called Android and the implementation really does use `android.graphics.RuntimeShader`, so why can desktop draw at all?

The source makes it clear. The library splits into three parts. commonMain holds all the logic in 2,270 lines, androidMain is 146 lines and skikoMain is 134 lines, and those last two handle the landing on each platform.

The shaders are AGSL strings. On Android they go into the system RuntimeShader. On the Skia side they go straight into `RuntimeEffect.makeForShader`, because AGSL was a subset of SkSL to begin with.

Which means the whole glass shader from Android compiles as it is once you hand it to Skia.

Desktop, iOS, and wasm all take this skiko path.

The support checks for both sides are in the source too. Android needs `SDK_INT >= 31` for RenderEffect and `>= 33` for RuntimeShader, while both functions on the Skia side just return true, with no threshold at all.

## Two things that do not go so smoothly

The first is in the docs but easy to miss: the lens effect requires that `shape` be a `CornerBasedShape`. I used `RoundedCornerShape`, so it was fine, but swap in a custom Shape and you have to handle it yourself.

The second is in the source and not mentioned in the docs: the Skia implementation does not support integer array uniforms. `setIntUniform(name, values: IntArray)` throws `UnsupportedOperationException` on desktop, while the Android side is fine. You will not run into this day to day, but the moment you write a custom shader you will.

A note on how the docs phrase things. The official API page says the effects are RenderEffect under the hood, in effect from Android 12, and anything involving RuntimeShader needs Android 13. That sentence does not hold in a desktop context. I ran lens on macOS with no trouble at all. It is written from the Android point of view, so do not read it as a platform limit.

## Performance, and the edges I hit

On this M series Mac, two pieces of glass animating together held a steady 60 fps, right at the refresh rate. Nine pieces of glass on screen at once, each with an animated lens, dropped it to 58.7 fps.

So at this scale, the cost of the effect has not become the bottleneck.

On the build side, the first run with dependency resolution took 25 seconds, and after that the Gradle cache keeps it under a second.

I should explain the screenshots as well. macOS would not give me screen recording permission, so system level capture only picked up the wallpaper. I switched to the `screenshot()` that ships with SkiaLayer, writing the window contents straight to PNG, and the pixels are accurate.

![](/images/liquid-glass/fig5.png)

## Where the community stands

Those 4,509 stars are not for nothing. The effect genuinely grabs people. But the pace is what it is: commits stopped on August 26, and the docs site lists its last update as 5 months ago.

There are 40 open issues. The first entry in the official FAQ is why the effect does not work, and the answer is that you probably forgot `layerBackdrop`. The second is more direct, an Android SIGSEGV crash that blows up in RenderThread, and you have to follow a tutorial to patch around it.

It is a good library, but it is one person's project, not an official component. Expect Compose Foundation levels of stability and you will be disappointed.

## My take

If you want macOS flavored glass panels on Compose desktop, this library works today, at a cost of two hundred KB and twenty lines of code.

Just do not use it as a finished component. It gives you the effect. Interaction, accessibility, and light and dark adaptation are all your own work. Apple's version looks good partly because the system deals with those corners for you.

I plan to try it in one of my small tools. What I really wanted to verify was that one thing. An Android native library, riding the shared bloodline between AGSL and SkSL, running on desktop like it belongs there.

Would you bring a personally maintained library into your cross-platform project for one visual effect?
