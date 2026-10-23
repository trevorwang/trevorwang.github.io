---
title: "Kotlin 2.4 Is Here and HarmonyOS Is Adapting to 2.4: Two Fronts for KMP"
date: 2026-09-18T12:34:38+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "HarmonyOS"]
summary: "A while back I wrote a piece on HarmonyOS and KMP, and at the time I found one key fact: the stable release the community published was HarmonyOS support built on KMP 2.2.21 / CMP 1.9.2."
cover:
    image: "/images/kmp-24-ohos/cover.png"
    alt: "Kotlin 2.4 Is Here and HarmonyOS Is Adapting to 2.4: Two Fronts for KMP"
    relative: false
---

## One branch name gives away what is happening

A while back I wrote a piece on HarmonyOS and KMP, and at the time I found one key fact: the stable release the community published was HarmonyOS support built on **KMP 2.2.21 / CMP 1.9.2**.

Going back to that repo the other day, I found one extra line in the branch list:

> **develop-2.4**
> Documentation set for framework KMP **2.4.21** _ CMP **1.12.0**

And the branch **was updated 21 hours ago**.

There is another one:

> **main-2.2** (default branch)
> Documentation set for framework KMP **2.2.21** _ CMP **1.9.2**

On one side, the stable release already shipped. On the other, the new baseline being pushed forward.

**On the HarmonyOS front, KMP is starting to line up with the latest upstream version.**

And at almost the same moment, upstream JetBrains had just released Kotlin 2.4.20.

Two tracks, colliding at one point in time.

## First, the upstream track: what changed in Kotlin 2.4.20

If you only skim the changelog you might think, oh, another patch-and-polish release — of the two hundred-odd changes, most are bugfixes.

But a few of them land hard for **people who write KMP**.

**One: Swift interop can finally hold its own.**

Swift export now supports mapping Kotlin's `sealed class` straight to a Swift `enum`:

```swift
// no more default
let name = switch shape.sealedType() {
case .circle(let t):    "圆"
case .rectangle(let t): "方"
}   // ← exhaustiveness check
```

Handling this kind of polymorphic type used to mean either a pile of `if let` on the Swift side, or being forced to add a `default` branch that can never run — **which kills exhaustiveness checking outright**. Now the compiler catches the gaps for you.

There is also **cross-language inheritance**: a Kotlin type can inherit from a Swift class or protocol (including pure Swift libraries), with no wrapper layer needed.

Both point the same direction: **the interop friction KMP has always been criticized for on iOS is being sanded down, bit by bit.**

**Two: klib incremental compilation hits Beta.**

```
kotlin.incremental.native=true
```

Change one line of Kotlin and you no longer wait for the whole native module to rebuild. If you have done Kotlin/Native, you know what that does for the day-to-day.

**Three: the Kotlin/JS test toolchain moved from Karma to Playwright** — about time.

**So the theme of the upstream track is: smoothing out the seams between languages.**

## Now the HarmonyOS track: where it actually stands

To understand the HarmonyOS side you have to start with its **version numbering scheme**, written in the stable release announcement that went out yesterday:

> Version numbers use "**upstream baseline + HarmonyOS adaptation sequence**": 2.2.21-1.0.0 / 1.9.2-1.0.0 means a HarmonyOS adaptation stable release based on upstream KMP 2.2.21 and CMP 1.9.2.

That rule matters. It says one thing:

> **HarmonyOS support is not a separate stove. It is chasing upstream versions.**

Which is why you see numbers like `2.2.21-1.0.0` — the `2.2.21` in front is the upstream Kotlin version it corresponds to, and the `1.0.0` behind is the HarmonyOS adaptation's own iteration number.

Already published to Maven:

| Component | Version | Status |
|---|---|---|
| kotlin | 2.2.21-1.0.0 | Published to Maven |
| compose-multiplatform | 1.9.2-1.0.0 | Published to Maven |
| compose-multiplatform-core | 1.9.2-1.0.0 | Published to Maven |
| skiko | 0.9.22.2-1.0.0 | Published to Maven |
| akinterop | 1.0.1 | Published to Maven |
| KMP OHOS Support | 1.0.0 | Published to the plugin marketplace |

And the new `develop-2.4` branch targets a baseline of **KMP 2.4.21 / CMP 1.12.0**.

The development cadence announcement says it too:

> **develop** branch: used for monthly development-state code merges, **merged into main after each monthly release gate**.

In plain language: **one version forward every month, merged into the stable line only once it is ready.** Not shipping on a whim, but a rolling cadence.

## What that branch has been busy with

I went through the `develop-2.4` commit history. The recent activity is interesting.

The latest one adapts the **CAPI user manual to API 26** — bumping the OHOS CAPI version to API26, updating "modules, minimum API, and typical dependency libraries" to match, and adding a whole section on **"special changes caused by the CAPI upgrade to API 26."**

Another one updates `akinterop` to 1.0.1 and `sqldelight` to 2.2.1-1.1.0.

**Those two directions map exactly to the two real blockers for HarmonyOS cross-platform work:**

- **CAPI follows the HarmonyOS system version** — the system upgrades, the bridge layer has to upgrade with it. That is the "do not fall behind" problem
- **Adapted version numbers for third-party libraries** — whether the ecosystem libraries keep up decides whether a project dares to use this at all

## While we are here: what CMP 1.12 brings upstream

Since `develop-2.4` is tracking CMP 1.12.0, what changed in 1.12 upstream is worth a look too.

CMP 1.12.0 shipped on August 25, and the highlight is on the Desktop side:

> MCP server was introduced for Compose Hot Reload that enables **AI agents to interact with a running Compose application in real time**

In plain language: **Compose hot reload got an MCP server, so an AI agent can interact with a running Compose app in real time.**

That one is subtle. The intersection of Compose and AI is moving from "use AI to write Compose code" toward "**let AI operate the running app directly**."

It also brings a batch of breaking changes, so watch out when migrating:

- The deprecation level on typealiases like `NativeCanvas` and `NativePaint` was raised to **ERROR**
- The `background` parameter on `SwingPanel` is deprecated

**Jumping from 1.9.2 to 1.12.0 is not just a version number change — migration cost counts too.**

## What the two things together tell you

Put the two tracks side by side and the picture is fairly clear:

**KMP's "one codebase, many platforms" is being pushed forward on several fronts at once.**

- **Upstream** is polishing cross-language interop (Swift export, incremental compilation)
- **The HarmonyOS community track** is chasing upstream versions (2.2 → 2.4), filling in system support (CAPI → API 26), and widening the third-party library ecosystem

They are not the same team, but they are heading in **the same direction** — making "write Kotlin once, run on more platforms" a little less friction.

For developers, the test is simple:

> **When you evaluate a new platform, the question is no longer "does it support this" but "which upstream version has it caught up to, are the libraries enough, does the performance hold up."**

**From "can it run" to "can it keep up" — that shift is itself a new stage.**

## But the community's real voice is worth more than the announcement

Announcements are always pretty. I went and read the community discussion forum on purpose, because that is where this track's actual temperature is.

**The hottest thread asks "can this be merged into the official JetBrains branch soon"** (started 30 days ago, 7 replies).

The poster was direct:

> Adding HarmonyOS to my existing pure KMP & CMP app project **runs into a large number of unsupported libraries**, and during compilation several already-adapted libraries show internal dependency conflicts. I hope it can be merged upstream soon so library developers across the whole KMP community can add ohos versions, **the CPF community's progress on library support is far too slow**.

He named three libraries specifically: `androidx-datastore`, `androidx-room3`, `androidx-sqlite` — "these three are basically the foundational dependency libraries for the persistence approaches everyone uses."

Then the part that stings:

> None of them support ohos **x86**. On Windows a developer can barely build and test an app with normal functionality — at minimum you have to keep a HarmonyOS phone plugged in as the test target...

He also gave a concrete trap: a `room3` alpha pulled from some mirror repo generates code that errors on a missing `androidx.sqlite` implementation, while upstream `room3 3.0.1` works fine.

**The official reply came, and it was honest:**

> Hello, **support for ohos x86 is already in planning**, thanks for your interest.

Other threads are each real in their own way:

- **"Under skia rendering the page frame production is dragged to about one frame per 3 seconds, suspected abnormality in the VsyncOnDemand deferred deregistration mechanism"** — performance
- **"material3 blocks porting an existing project"** — component library traps
- **"Is webView supported right now?"** — capability coverage
- **"Has anyone actually shipped a project? What traps did you hit"** — and someone replied with: **"currently doing a KMP adaptation of a voice assistant"**

**Add these up and you have the real state of this track: it runs, people are using it, but the ecosystem and performance are still catching up.**

## Why "following upstream" is slow work

Seeing that "progress is far too slow" line, some people will assume it is a matter of insufficient investment.

But there is a technical reality worth spelling out: **HarmonyOS support changes the compiler backend. It is not writing a plugin.**

Go through the CPF-KMP-CMP repo list and you see these names:

`kotlin`, `compose-multiplatform`, `compose-multiplatform-core`, `skiko`, `akinterop`, `cicd-toolkit`…

**These are HarmonyOS forks of the Kotlin compiler, the Compose framework, and the graphics rendering layer.**

Which means the work includes at least:

- Adding a new **compilation target** to Kotlin/Native
- Wiring Compose's rendering pipeline into HarmonyOS's **Render Service**
- Getting the Skia graphics library that Skiko wraps to run on HarmonyOS

**And every time upstream ships a new version, all of those changes have to be re-aligned.**

That is what the "upstream baseline + adaptation sequence" version scheme really means — **it describes the nature of the work precisely: continuous tracking, not a one-time port.**

Look again at "monthly development-state merges, merged into main after each monthly release gate" and it reads differently:

**This is not slow. This is doing a very heavy thing at a pace that can be sustained.**

## A few of my own views

**One: the essence of "adaptation" here is following upstream, not reinventing the wheel.**

You can see it in the version scheme alone (upstream baseline + adaptation sequence). The core work on the HarmonyOS track is getting upstream's compiler and UI framework running on HarmonyOS, not inventing a separate API.

**Which means its ceiling is tied to upstream**: upstream smooths out Swift interop and that is a dividend for iOS; upstream improves the compiler and HarmonyOS only gets it by keeping up.

**Two: the three community complaints point at the same problem.**

x86 support, third-party library versions, performance — three things on the surface, all **"ecosystem maturity"** underneath.

And ecosystem maturity has a cruel property: **it cannot be solved with enthusiasm, only with time and real projects wearing it in.** The official reply of "already in planning" is true, and there is a certain helplessness in how true it is.

**Three: if you are evaluating whether to get on this track, here is a framework.**

Do not ask "does it support HarmonyOS" — **ask these three questions**:

1. **Which upstream version has it caught up to?** (right now 2.2.21 is stable, 2.4.21 is in development)
2. **Have the libraries I depend on been adapted?** (especially foundational ones like persistence and networking — `room3` and `datastore` are the acknowledged pain points)
3. **What is my dev machine?** (if you need an x86 emulator, you still have to wait; an Apple silicon Mac plus a real device is the steadier combination)

**All three clear, then start. One blocked, and the plan goes in a note for later.**

## Last

Kotlin 2.4 and HarmonyOS 2.4 look like two unrelated things.

But they are answering the same question:

> **One body of Kotlin code: how many places can it run, and how well?**

Upstream is sanding down the seams between languages, and the HarmonyOS track is widening the map.

**Both are making that number bigger.**

---

I want to ask you something:

**Would "HarmonyOS support has caught up to 2.4" make you reconsider a KMP project you have on your desk?**

**Or have you already been through the traps on this track?** Talk in the comments — I would genuinely like to hear from real projects.

---

*References: CPF-KMP-CMP community repos (AtomGit/GitCode) branch and release announcements, public posts in the community discussion forum, JetBrains Kotlin 2.4.20 release notes.*
