---
title: "A Patch That Saved 15 MiB Got Rejected, and the Reason Was Not in the Code"
date: 2026-10-09T08:10:57+08:00
draft: false
categories: ["WeChat"]
tags: ["Tech"]
summary: "An engineer was doing size optimization on his own iOS build and found that one extra compiler flag shrank the bundle by 15 MiB. He filed an issue with a reproduction on a public project and data from real devices, and opened a pull request that same night."
cover:
    image: "/images/kn-outliner/cover.png"
    alt: "A Patch That Saved 15 MiB Got Rejected, and the Reason Was Not in the Code"
    relative: false
---

An engineer was doing size optimization on his own iOS build and found that one extra compiler flag shrank the bundle by 15 MiB. He filed an issue with a reproduction on a public project and data from real devices, and opened a pull request that same night.

The next morning the PR was closed. No technical review.

The reason given: they do not accept low-effort AI-generated contributions. To feel the weight of that sentence, you have to read the technical part first.

## What he found: small-binary mode never touches Kotlin code

The subject is `smallBinary`, an experimental option in Kotlin/Native since 2.2.20. It switches the IR pipeline's optimization level to Oz and stops forcing inlining of runtime hot paths. Sounds like everything that should be cut gets cut.

But for the final code generation step, Kotlin/Native still runs `clang -cc1 ... -O3`. That value lives in the toolchain's `konan.properties` under the key `clangOptFlags.ios_arm64`.

And LLVM's MachineOutliner, the pass that removes duplicate machine code, only applies by default to functions tagged with the `minsize` attribute on AArch64.

Stack the two: the IR layer is in size mode, the machine code layer is still in performance mode, and functions Kotlin generates carry no `minsize`. The result is that MachineOutliner sweeps through and touches zero Kotlin functions.

He put two sets of numbers in the issue. Of the 33,881 function definitions in the final bitcode, only 55 carry `minsize`, all of them runtime cold paths. In their app, the object holding 33 MB of Kotlin code has 11 outline helpers; the precompiled Skia object next to it has 4,459.

## I ran it myself

You should not take someone else's numbers on something like this. I had Kotlin/Native 2.4.0 locally, wrote a small chunk of interface-dispatch-heavy code, 300 functions, and compiled the same source file three times, all with `-opt` and `-Xbinary=smallBinary=true`.

![](/images/kn-outliner/terminal.png)

The mechanism reproduced completely. By default there are only 9 outline helpers, which shows the pass really is skipping Kotlin code. Add one konan.properties override and the helper count goes into the thousands, with size dropping in step. Absolute size only fell 5% to 9%, because my sample project is tiny while his Kotlin object is tens of MB.

My local konan.properties does indeed contain `clangOptFlags.macos_arm64 = -O3` and `llvmInlineThreshold = 100`, so both premises of his argument check out.

## The size saved, and the other side of the ledger

His app is a Compose Multiplatform iOS build, unsigned archive at 114.7 MiB (same commit, Kotlin 2.4.10, Xcode 26.5).

```
smallBinary + latin1Strings         114.7 → 101.7 MiB
plus forcing the outliner on         → 86.9 MiB
plus inline threshold 5              → 81.4 MiB
```

That 81.4 MiB in step three deserves a closer look: he built 81.3 MiB using a patched compiler that tags every Kotlin function with minsize, and the two differ by 0.1 MiB. In other words most of that gain comes from less inlining, and you can get it without touching the compiler.

He also spelled out the costs. A 23-second scripted scroll on iPhone 13 pushed CPU from 8.48s to 8.94s, up 5.4%, and janky frames from 18 to 27; on iPhone X CPU rose 6.5%. Startup time unchanged.

The counterintuitive one is download size. The whole executable, gzipped, went from 20.42 MB to 20.05 MB. It moved 0.37 MB. The unsigned bundle lost 14.8 MiB and compressed almost not at all, because the duplication being removed is exactly what a compression algorithm likes to eat.

Also, framework code generation time went from 2 minutes to 6. And the top of a crash stack gains an `OUTLINED_FUNCTION_n` frame, though lldb still walks back to the real caller — he tested Kotlin exceptions and SIGSEGVs on both builds and the logical stack matched.

His own situation was a finance app, and his conclusion was that he would rather have smooth scrolling than save those 15 MiB. The two binary options were in review; the outliner one waits until his heaviest page is device-tested.

## Then JetBrains closed his PR

Technical part done. Now the rejection.

> We don't accept low-effort AI-generated contributions that shift the burden of understanding, verifying, and fixing the proposed changes onto maintainers.

In other words: they do not accept low-effort AI-generated contributions that move the burden of understanding, verifying and fixing onto maintainers. There was a follow-up line too: using AI tools is fine, but contributors have to put in their own engineering effort and take responsibility for the quality and correctness of what they submit.

His own retrospective in the post is candid. He admits he used AI tools to prepare the PR, opened it before the team had looked at the issue, then rewrote it an hour later. From his side it was one rushed move; from the maintainer's side it was the kind of drive-by submission they get buried in every day.

He does draw one boundary: the PR being closed does not overturn the technical conclusion. Those numbers reproduce with one flag and a stock compiler, and the public KotlinConf app gives anyone an independent way to check. Worth mentioning: that public app's Kotlin object went from 14.88 MB to 8.33 MB.

The issue is now marked Declined, created October 7, updated the next day.

## What people writing Kotlin/Native can do now

If your project ships an iOS framework, do these three things in order.

First, turn on the `smallBinary` and `latin1Strings` binary options. They work with a stock compiler, both are experimental, and remember to retest on every Kotlin upgrade. That step got him 13 MiB.

To push further you go down the konan.properties override path: force the outliner on, push the inline threshold down. It does pay off, but it is undocumented usage. What is officially documented is `-Xllvm-module-passes` and `-Xllvm-lto-passes`, and KT-69212 says you cannot currently disable a specific pass outright.

That kind of override has a hidden risk: if a release renames the property, it silently stops working. So put a size check in CI.

Finally, always device-test performance before shipping. This optimization trades CPU for size, and his measured scroll cost starts at 5% and up.

A few items he also tested that did not pan out:

- A vendor's Swift package pulled in a 4.9 MiB OpenSSL framework that was never used; check with `otool -L` plus `nm`
- The ICU data Skiko ships is 6.3 MB; he trimmed a build and long-pressing the input field crashed outright, so he stopped there
- `material-icons-extended` is declared in several modules but not one byte of it reached the bundle; removing it only saves build time
- `@Preview` functions are stripped in release anyway
- LLVM's famous global function merging pass actually made the Kotlin object 0.44 MB larger under his configuration

## Last

What is interesting here is not those 15 MiB. A reproducible performance problem, a public reproduction project, a set of on-device measurements — with all of that assembled, what stopped it was not a technical dispute. It was the trust cost inside the contribution process.

For people who write code it is a reminder: let maintainers evaluate the issue first, do not rush to open the PR. For maintainers it is a different question: AI has pushed the cost of producing a PR to nearly zero while review cost has not dropped at all. More projects are going to draw this line.

Would you take a 5% scroll cost to save 15 MiB? Tell me in the comments.

References:
- Mo Yaver's detailed write-up https://medium.com/@mhammadjaber00/kotlin-natives-smallbinary-never-runs-llvm-s-outliner-on-kotlin-code-f457d0fbb6d2
- The issue he filed, KT-90068 https://youtrack.jetbrains.com/issue/KT-90068
- The closed PR #8802 https://github.com/JetBrains/kotlin/pull/8802
- Kotlin 2.2.20 release notes (smallBinary and latin1Strings) https://kotlinlang.org/docs/whatsnew2220.html
- Kotlin/Native binary options docs https://kotlinlang.org/docs/native-binary-options.html
- Custom LLVM optimization passes https://kotlinlang.org/docs/native-llvm-passes.html
