---
title: "JetBrains Pulled Swift Support Out of the KMP Plugin: Not One Link in the Toolchain Moved"
date: 2026-10-06T10:44:56+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "On October 5 JetBrains published a short announcement with a blunt title: IDE support for the Swift language is ending in the KMP plugin. The effective versions are specific: IntelliJ IDEA 2026.3 and Android Studio Rabbit 2 | 2026.2.2."
cover:
    image: "/images/kmp-ios/cover.png"
    alt: "JetBrains Pulled Swift Support Out of the KMP Plugin: Not One Link in the Toolchain Moved"
    relative: false
---

## First, what the announcement literally says

On October 5 JetBrains published a short announcement. The title is blunt: IDE support for the Swift language is ending in the KMP plugin. The effective versions are specific: IntelliJ IDEA 2026.3 and Android Studio Rabbit 2 | 2026.2.2.

The list of removed features is specific too. Copying it:

```text
This includes editing, syntax highlighting, and navigation within Swift
files and between Swift and Kotlin.
```

> In other words: editing, syntax highlighting, and navigation inside Swift files, and between Swift and Kotlin.

Reading those lines, a lot of people's first reaction is that KMP on iOS is finished. Hold on. The second paragraph of the same announcement draws the boundary.

## Which layer got removed

The second paragraph reads:

```text
Running iOS applications and debugging Kotlin code on iOS will remain fully
supported. Kotlin/Native functionality, such as Swift Export and importing
Swift packages into Kotlin projects, is not affected.
```

> In other words: running iOS apps and debugging Kotlin code on iOS stay fully supported. Kotlin/Native features like Swift Export and importing Swift packages into Kotlin projects are unaffected.

So what got taken away is the layer in the IDE that reads and edits Swift. The compiler, the Native toolchain, the path into Xcode are all still there.

I verified that second half on this machine. I already had the prebuilt Kotlin/Native 2.4.0 and Xcode 26.2, so I used them to compile a small piece of code into an iOS framework:

```bash
kotlinc-native -target ios_arm64 -produce framework -o out/shared src/shared.kt
```

Nine point eight seconds. The output is a Mach-O 64-bit arm64 dynamic library plus a header file for Xcode. Not one step in that chain regressed. Pulling the Swift features did not touch it.

![](/images/kmp-ios/ios-framework.png)

## So what actually carries iOS

Take the chain apart and the order is:

1. Kotlin/Native compiles the shared code into a framework Apple can use — that is the nine point eight seconds above.
2. Xcode links that framework into the iOS app. `embedAndSignAppleFrameworkForXcode` in Gradle is exactly this job.
3. The Swift shell calls Kotlin. Today that mostly goes through Objective-C header interop, and the team is separately pushing Swift Export, still in Alpha, aimed at letting Kotlin export directly as a Swift module and dropping the header layer.
4. Reading and editing Swift in the IDE was only convenience during development. Remove it and the three layers above do not move.

Get these four straight and the weight of the announcement becomes clear.

## Why they pulled it

The official reason is usage. Their telemetry and user research show that iOS developers on KMP projects mostly write Swift in Xcode, while the people working inside the KMP plugin increasingly use Kotlin and Compose Multiplatform.

There is also historical baggage: this Swift tooling was originally built for AppCode, and AppCode was discontinued long ago. Then it moved to Fleet, then into the KMP plugin. Keeping its core architecture in step with the continuous changes in Swift, Xcode and the IntelliJ platform is a real investment. They weighed it and decided not to, and they state plainly that they do not currently plan to replace it with something else.

That trade-off is actually on the same line as KMP's moves these past months: they are spending effort making the Kotlin side better, not doing Xcode's job for it.

## The community side is ahead of the secondhand reports

On the same day there was another item on the trending list, pointing the opposite direction. The iOS version of the open-source OSM editor StreetComplete entered TestFlight — the first testable build after years of porting work.

That port dates from a tracking issue opened in December 2023, still open today, 51 comments accumulated. The maintainer says a full port is roughly one person-year of work, that the first half of 2024 was pushed full-time on Prototype Fund money, and that by September of that year about half was done.

One claim in the secondhand reports has already expired. It says the Compose Multiplatform MapLibre library does not exist yet and is the blocker for the port. I went and read StreetComplete's build file. It currently says:

```kotlin
implementation("org.maplibre.compose:maplibre-compose:0.19.0")
```

That library has shipped on Maven Central from 0.11.0 all the way to 0.19.0. That gap is closed.

The costs have to be stated too. The prototype fund ran out; the maintainer writes in the issue that follow-on funding does not look good — OpenStreetMap has no money, NLNet's program is contracting, and the crowdfunding sponsors have not increased. Known issues are still open: the map does not work on iOS 15.x, forms can only be dismissed with a gesture, and there are still plenty of bugs.

## I counted the repo

While I was there I counted the repo by source set. More convincing than reading news:

- commonMain: 1,281 Kotlin files, 3.6MB
- commonTest: 287 files, 1.3MB
- androidMain: 34 files, 43KB
- iosMain: 33 files, 42KB

So each platform gets a shell of a bit over forty KB and the same app runs on iOS and Android. Among the 33 files on the iOS side, the biggest is IosModule.kt at six thousand bytes, doing dependency injection wiring; the rest are platform-specific pieces: networking, sync, upload, periodic cleanup.

![](/images/kmp-ios/source-sets.png)

See that ratio clearly and the blast radius of the announcement is clear too: the removed features act on that forty-two KB shell, and have nothing to do with the shared three point six MB.

## These two stories are one story

JetBrains's choice is "I do the Kotlin side well, the Swift side goes to Xcode". StreetComplete's approach fits that division exactly: shared logic and UI are all written in Kotlin and Compose, and the Swift side is left as a shell.

So the call is simple. Ask yourself a few things:

- Where do you actually write Swift in this project? If Xcode was open anyway, this announcement barely touches you.
- If you read and edit Swift inside Android Studio, you need to change your workflow from now on, or stay on an older version.
- Do not look at IDE support and toolchain support as the same thing. The first is gone. The second is not.
- A viable technical path does not mean a project reaches the finish line. The StreetComplete story is a reminder about money and people.

## How long does the version on your machine still work

The threshold is Android Studio Rabbit 2 | 2026.2.2 and IntelliJ IDEA 2026.3. Mine is 2026.2, so the Swift features you have are still there, they just stop getting updates. If you need them badly, upgrade a little later. If not, let it ride — Swift's real home has always been Xcode.

In your KMP project, where do you write the Swift on the iOS side? Comments are open.

References:
- https://blog.jetbrains.com/kotlin/2026/10/discontinuing-swift-language-ide-support-in-the-kotlin-multiplatform-plugin/
- https://github.com/streetcomplete/StreetComplete/issues/5421
