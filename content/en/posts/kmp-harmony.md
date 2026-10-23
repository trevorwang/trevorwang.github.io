---
title: "KMP Lands on HarmonyOS: One Compose Codebase, Four Platforms Now"
date: 2026-09-16T21:26:40+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "HarmonyOS"]
summary: "Start with a number: four."
cover:
    image: "/images/kmp-harmony/cover.png"
    alt: "KMP Lands on HarmonyOS: One Compose Codebase, Four Platforms Now"
    relative: false
---

## A quiet bit of "map expansion"

Start with a number: **four**.

Android, iOS, desktop — that's the main ground Kotlin Multiplatform has covered steadily for years (Web / Wasm is still in progress). Write one set of shared logic, add one Compose UI, and several platforms run together.

And yesterday, that list gained a fourth member: **HarmonyOS**.

And this time it isn't "theoretically runs" — it's an **official release**. Components are in the Maven repo, the toolchain plugin is searchable in the Android Studio plugin marketplace, and the docs are on AtomGit.

What surprised me most was the publisher's line:

> Teams already using CMP on other platforms can reuse the same Compose code for UI and business logic directly, without writing a whole separate ArkUI implementation for HarmonyOS.

**No ArkUI rewrite.** For anyone doing cross-platform, those few words carry real weight.

## Let's get the facts straight

What shipped is the **first HarmonyOS release of KMP&CMP**, on September 15, 2026.

The key facts:

- **Who made it**: built jointly by community partners in the CPF-KMP-CMP SIG (note: a community SIG, not JetBrains official — I'll get into that distinction later)
- **Based on**: upstream **KMP 2.2.21 / CMP 1.9.2**
- **Goal**: bring the "shared logic, native UI rendering" paradigm to HarmonyOS
- **How to get it**: components are published to the Maven repo; the toolchain plugin **KMP OHOS Support** is searchable and installable from the Android Studio plugin marketplace

The official positioning on who it's for is clear too:

> For teams already using KMP/CMP for cross-platform work, this release helps lower the cost of moving to HarmonyOS. HarmonyOS teams can also use it for mixed Kotlin and ArkTS development, and to widen their technology options.

Translated: **KMP teams who want HarmonyOS now face a lower cost; HarmonyOS teams who want Kotlin now have a path.**

Some background while we're here — this has been building for three months. The community edition opened Beta in June, and in August InfoQ reported a round of measurements (95% less rendering memory, 90% fewer GC jank incidents). Now it's properly landed.

## Highlight one: Compose runs on HarmonyOS directly, no UI rewrite

This is the one I think is most valuable.

On HarmonyOS, the official route is **ArkTS + ArkUI**. For a team already doing KMP/CMP, that means the HarmonyOS side needs **its own separate UI** — business logic shared, but the interface rewritten.

This release paves that road:

> CPF-KMP-CMP provides platform implementations of compose.ui, compose.foundation, compose.material and compose.material3 on the HarmonyOS side, so you can keep the familiar declarative UI habits while building HarmonyOS screens.

**compose.material3 is there.** Which is to say, the Compose code you're used to writing is recognized on HarmonyOS.

For a project already running CMP on Android/iOS, that means **the cost of adding HarmonyOS drops from "rewrite a UI" to "add a target"**.

How big a difference is that? Anyone who's done cross-platform knows.

## Highlight two: Kotlin ↔ ArkTS two-way interop

The second one matters a lot for existing projects.

The reality is: plenty of HarmonyOS projects **already have a pile of ArkTS** (system capabilities, business modules), and ripping it all out for Kotlin isn't an option.

The answer here is **akinterop**:

> Shared logic can keep being written in Kotlin. When the system side or existing modules stay in ArkTS, use akinterop for two-way interop: declare intent with annotations / decorators, let the toolchain generate the bridge, and write less C glue. Primitive types, containers and exceptions can cross sides, existing projects can adopt incrementally, and nothing has to be torn down for interop.

**"Adopt incrementally, nothing torn down"** — that line is the reassurance existing projects needed.

Annotation declares intent, toolchain generates the bridge — a far better experience than hand-writing C glue. Anyone who's written JNI knows that pain.

## Highlight three: unified rendering (this is the technical core)

This one's more hardcore, but I think it's the most technically substantial piece of the whole thing.

The problem first: **traditional cross-platform UI approaches usually build their own rendering pipeline.**

What's wrong with building your own? The report says it directly:

> Traditional cross-platform UI solutions often build their own rendering pipeline, which tends to add initialization and composition overhead, and often shows up as slow startup and animations that don't match the system.

Self-rendering means your UI doesn't go down the system's path — startup needs an extra init step, animation rhythm doesn't line up with the system, and it just looks "not native".

This approach goes **the other way**:

> CPF-KMP-CMP connects Compose's drawing capability to HarmonyOS's Render Service / RenderNode system rendering, with HarmonyOS handling composition and refresh scheduling uniformly (unified rendering requires API 19 or above).

**Handing drawing over to HarmonyOS to composite itself** pays off in a chain:

- **Faster first frame**: no self-rendering init wait, so startup and navigation feel snappier
- **Fine-grained partial refresh**: update only when state changes, with partial recording support — **only the changed region is repainted**
- **Native component mixing**: Compose screens can mix with ArkUI and C-API, so native capabilities like maps and video are easier to embed
- **System-aligned animation**: frame rhythm follows the system's DisplaySync, so animation matches native HarmonyOS apps

That last one matters especially. The reason cross-platform apps look fake at a glance often isn't wrong layout — it's that **the animation rhythm is wrong**, frame scheduling off the system's beat.

> ⚠️ Note one limitation: **unified rendering requires API 19 or above**. On older devices it degrades.

## The performance account: 20% off compile time, 30% off heap

They gave some genuinely specific numbers this time, unlike the vague claims that usually pass for "performance optimization":

**Compile side (parallel compilation):**

> Splitting LLVM IR and running local optimization across threads before merging shortens compile waits on large projects; typical scenarios save roughly 20%+ of the time.

**Memory side (CommonGC):**

> Using a concurrent mark-copy collection algorithm to reduce memory fragmentation and GC pause time, KN heap usage in typical scenarios drops by about 30%.

There are also two engineering optimizations that are genuinely practical:

- **Modular splitting**: a single KMP project can be configured into multiple `.so` files, and combined with distributed compilation it shortens builds further
- **Debug incremental compilation**: on macOS / Linux, incremental compilation based on a static library cache — **only the changed part recompiles, no full relink every time**

(Anyone doing KMP probably has a small stress reaction to the words "full relink")

One more detail I like — **no idle spinning in the background**:

> The unified rendering path supports requesting VSync signals on demand, actively suppressing refresh drawing when page content hasn't changed; it also supports LTPO hardware characteristics, stopping pointless draw loops automatically once the app goes to the background... This is framework default behavior, requiring no extra code from the app.

**Default behavior, no app code needed.** This kind of "the framework worries about it for you" design is far friendlier than making you turn animations off by hand in `onPause`.

## Getting started: three steps

The flow is simpler than I expected:

> (1) DevEco Studio: use 6.0.0+ to download HarmonyOS SDK API 17+, open the harmonyApp in the project and set up the debug signature.
> (2) Android Studio: write KMP&CMP code day to day.
> (3) Android Studio: pick a device and harmonyApp, hit Run (keep DevEco running while pushing the package).

**Note the parenthesis in step 3** — DevEco Studio has to stay open while pushing. That kind of "two IDEs cooperating" detail is exactly where people trip, and they said it up front.

The docs are on **AtomGit** (search `CPF-KMP-CMP/docs`) if you want to dig in.

## Why this piece of the HarmonyOS puzzle matters so much

Honestly: for KMP developers in China, this puzzle piece may matter far more than Web/Wasm.

The reason is practical — **for mobile work in China, HarmonyOS is no longer a "should we consider it" question, it's a "when do we have to" question.** Especially in government and enterprise, finance, automotive and IoT, where it's moving faster than many expected.

And the choice KMP teams faced with HarmonyOS used to be awkward:

- **Shared logic carries over, but the UI gets rewritten** (ArkUI) → which means an extra person on the HarmonyOS side
- Or **drop KMP entirely** and start a separate HarmonyOS project → all the shared-logic advantage gone

The point of this release is right there: **it takes most of the "HarmonyOS UI has to be rewritten too" cost out of the equation.**

For a small team, that may be the difference between "can't afford it" and "can afford it".

## A cooler read: how to see this

Hype is hype. I need to be clear about a few boundaries so you don't misjudge.

**First, this is a community SIG project, not JetBrains official.**

The publisher is the **CPF-KMP-CMP SIG**, not JetBrains and not an official Huawei release. That doesn't make it bad — quite the opposite, this kind of "ecosystem foundation" usually gets chewed through by community SIGs bit by bit — but **set your expectations accordingly**:

- Long-term maintenance depends on community activity, without the explicit commitments an official API carries
- When you hit a problem, the answer is in the community (issues, discussions), not in official docs

**Second, unified rendering needs API 19+.**

If your target devices include a lot of older HarmonyOS builds, you don't get that biggest performance dividend, so assess your device distribution.

**Third, it solves a real problem.**

Cross-platform on HarmonyOS used to offer basically two options:

- **Write ArkUI separately for HarmonyOS**, maintaining two UIs
- **Skip HarmonyOS**, wait for official support

Now there's a third: **keep writing shared logic, reuse the Compose UI, bridge existing ArkTS with akinterop.**

For a team that's already invested in KMP, the cost-benefit of that third road is hard to ignore.

## Put it back in the arc of the last few days

Funny thing: this lines up with two topics from the last couple of days into one thread.

In **the Jake Wharton piece**, he puts it clearly:

> Knowing Kotlin is no longer a career advantage... but having something more advanced like Kotlin Multiplatform can still be a differentiator.

**The Shopify piece** concludes: once AI makes "reimplementing it" cheap, the "share the UI" camp takes the biggest hit, while "shared logic + native UI" holds up better.

And then today: **KMP's map just gained another region — and a big one.**

Three things point the same way: **the value of cross-platform is shifting from "save headcount" to "architecture governance + ecosystem coverage".**

Chewing through platforms one at a time was always the core story of the KMP bet. With the HarmonyOS piece filled in, for developers in China especially — **it may be the first time many of them have a real reason to look at KMP seriously.**

---

One question to leave you with:

**If you can write HarmonyOS UI in Compose, would you still maintain a separate ArkUI codebase for it, or just extend your KMP project to HarmonyOS?**

And — for the project in your hands, is HarmonyOS a "must do" or a "let's see"?

Comments are open, I really want to know where people actually are.

---

*References:*
- *[First official HarmonyOS release of KMP&CMP (Kuai Technology / Drive Home, 2026-09-15)](https://news.mydrivers.com/1/1151/1151178.htm)*
- *Docs: search `CPF-KMP-CMP/docs` on AtomGit*
