---
title: "M6 Benchmarks Match M3 Max — Is KMP Desktop Worth Betting On?"
date: 2026-09-20T08:33:10+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "Apple's M6 benchmarks have been trickling out the last couple of days. Single-core about 24% above the M4 part, multi-core about 48%, and multi-core straight up matches M3 Max — a mid-range chip tying the previous flagship on multi-core is genuinely wild."
cover:
    image: "/images/m6-kmp-desktop/cover.png"
    alt: "M6 Benchmarks Match M3 Max — Is KMP Desktop Worth Betting On?"
    relative: false
---

## The benchmarks are genuinely wild

Apple's M6 benchmarks have been trickling out the last couple of days. Single-core about 24% above the M4 part, multi-core about 48%, and multi-core straight up matches M3 Max — a mid-range chip tying the previous flagship on multi-core is genuinely wild.

On the GPU side, 12 cores pulled 4,071 single-core and 22,783 multi-core on Geekbench, roughly 20% above M5.

And then I started seeing people discuss it: chips are this fast now, should we build out desktop too?

So I tried it on a project of my own. The conclusion is a bit of a letdown: this has basically nothing to do with chip benchmarks.

## What I actually ran

The machine is a Mac mini, M4, 10 cores, 16G of RAM. The project is a KMP + Compose Multiplatform AI chat client, with shared holding the shared logic and desktopApp as the desktop entry point.

Packaging is one command:

```
./gradlew :desktopApp:createDistributable
```

Ran it twice: 22 seconds the first time, 8 the second. Both fast — but dependencies were already downloaded by then and the code hadn't changed much. The real first cold build (fetch dependencies, compile Kotlin, generate the runtime image) is measured in minutes.

Fast is fast, but that's not the point. The last number is:

```
$ du -sh lange.aichat.app
133M
```

One hundred and thirty-three megabytes. For a chat client.

It gets interesting when you break it open:

| Directory | Size | What's in it |
|---|---|---|
| Contents/runtime | 77M | The bundled JRE |
| ├ lib/modules | 52M | JRE core modules |
| └ libjvm.dylib | 17M | The JVM itself |
| Contents/app | 55M | App and dependencies |
| ├ libskiko-macos-arm64.dylib | 20M | Skia rendering engine |
| ├ skiko-awt-runtime | 8.9M | Skiko's JVM bindings |
| ├ material3-desktop | 4.4M | |
| ├ foundation-desktop | 4.4M | |
| └ ui-desktop / runtime / kotlin-stdlib | 3.3 / 1.9 / 1.8M | |

77M is the JVM itself, plus 20M of Skia, so the floor just to make the thing run is 97M. The part you wrote yourself is maybe a few hundred K. And that 77M is already after one pass through jlink, not the raw size.

So the account is clear: Compose Desktop is pleasant to write, but the distribution cost is sitting right there. Unless you replace the JVM premise (the GraalVM native-image road), a hundred-odd megabytes is the norm.

## If M6 is this much faster, where does it actually help

Compilation. Kotlin compilation, Gradle parallel tasks, native linking — these are all multi-core work, and 48% more multi-core genuinely shortens build times.

I also measured another target in the same repo while I was at it:

```
./gradlew :app:webApp:wasmJsBrowserDistribution
→ BUILD SUCCESSFUL in 4m 10s
```

4 minutes 10 seconds, all 50 tasks run, CPU and memory pegged. For anyone running several builds a day, this is the multi-core gain you feel most directly.

And end users?

For a chat client, the bottlenecks users can feel are these: cold start — JVM loading plus Compose init, which is single-core and IO, so multi-core barely helps; scrolling and animation — that's Skia, and 20% more GPU is overkill for a 2D UI; first screen — that's the network.

M6 could be twice as fast and users still wouldn't feel your app get faster.

## So should you build the desktop app or not

Here's how I see it.

If you already have a KMP project and want one more platform, just add it. The shared logic is already there, it's one more entry point. Internal tools and developer tools too — you're only installing it on your own team's machines, nobody cares how big the bundle is.

What genuinely deserves its own project is a different class of need: reading and writing files locally, multiple windows that stay resident in the background, running models on the machine. Those are things desktop can do and phones can't. In that case 133M is nothing — users are installing a tool, not a toy.

Conversely, be careful with lightweight consumer utilities. Nobody downloads 133M for a little weather app. Same for hard cold-start requirements: you can't route around the JVM cold-start curve, unless you go GraalVM.

Side note, Compose Multiplatform 1.13 is already working on AppCDS and AOT (the 1.13.0-alpha01 notes mention "significantly faster app startup"). If you're hesitating right now because startup is slow, you can wait one or two releases.

## Last

M6 is great. Builds get faster, and for people typing code every day that's a real gain.

But it doesn't change the desktop trade-off. What holds you back is bundle size and cold start, and the chip has nothing to do with either. If your motivation is "capture the new hardware dividend", you may be looking in the wrong place — the real desktop dividend is in packaging technology (jlink, AppCDS, GraalVM), not CPU benchmarks.

If you're already doing KMP, desktop is actually the cheapest end to add: shared is written, add an entry point and you have one more platform. That reason has nothing to do with chips, and I think it's more solid than a benchmark score.

Would you add desktop to your project? How big a bundle can you live with? Comments are open.

---

*Test environment: Mac mini (Apple M4, 10 cores, 16GB), macOS 27.0, JDK 21.0.10, Kotlin 2.4.0 + Compose Multiplatform 1.11.x. M6 scores come from leaked Geekbench results and public reporting from cnBeta, Notebookcheck, Phoenix Tech and others.*
