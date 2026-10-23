---
title: "Building My Own PikPak Client with KMP: Not a Single expect/actual"
date: 2026-09-27T08:30:33+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "A couple of days ago a post about a niche piece of software crossed my feed — a project called Piko, posted by its own developer: an open-source third-party PikPak client, Android and Windows, sharing both UI and business code across the two."
cover:
    image: "/images/kmp-piko/cover.png"
    alt: "Building My Own PikPak Client with KMP: Not a Single expect/actual"
    relative: false
---

A couple of days ago a post about a niche piece of software crossed my feed — a project called Piko, posted by its own developer: an open-source third-party PikPak client, Android and Windows, sharing both UI and business code across the two.

Cloud drive clients are nothing new, but this year I have heard one codebase, two platforms so many times I've gone numb to it. So what actually made me stop was not that it runs on two platforms. It's the shape it gave that idea.

I cloned the repo and counted, and then I got stuck on one thing.

## The structure first

Piko is Kotlin Multiplatform plus Compose Multiplatform, five modules. The measured code volume looks like this:

- `shared` state and business logic, 79 files under `commonMain`
- `ui` shared interface, 74 files under `commonMain`, **this is the only source set it has**
- `app` Android entry point, 22 files
- `desktopApp` desktop entry point, 29 files
- `cli` a command-line tool for development, not shipped

![Piko module and source set layout](/images/kmp-piko/01-modules.png)

The first two add up to 153 files and over 31,000 lines of Kotlin, and that's the part both platforms share. The last two are the entry points, 51 files and 8,000-odd lines together.

So far this looks normal. What's not normal is the next command I ran.

## Not a single line of expect/actual

In KMP, the textbook answer for platform differences is expect/actual: write `expect fun` in common code, write `actual fun` in each platform's source set. I meant to count how many it used so I could quote it as an example.

```
grep -rn "^\s*expect " --include="*.kt" shared ui app desktopApp
expect declarations: 0
actual implementations: 0
```

I assumed my regex was wrong and tried a few variations. Same result. A project with 31,000 lines of shared code and 46,000 lines of Kotlin repo-wide uses expect/actual zero times.

What's stranger is the source set layout. `shared` has no `androidMain`, `ui` doesn't even have `desktopMain`, and the one platform-specific source set in `shared` is called `jvmSharedMain` — one file, 67 lines.

So this isn't "used rarely". It's a completely different approach.

## Its answer: interfaces, plus runtime injection

The author wrote one line in his architecture notes that I think is worth more than any paragraph in a tutorial:

> Platform differences go through interfaces, not expect/actual

The reason is in a comment sitting on the capability interface definition:

> The implementations need a Context, a window, and each platform's own playback backend. These are runtime objects, not compile-time platform differences.

That line cut through a muddle I'd been carrying. expect/actual solves compile-time differences: same function name, one implementation per platform, the compiler picks. But the things a real client needs to split are almost all runtime objects. Android needs a Context, needs to register ActivityResult launchers, needs libmpv's SurfaceView for playback. Desktop needs a window handle, needs MPV's native DLL, needs the video surface zero-copied into Skia. What these share is that at compile time they look like the same kind of thing. Only the construction differs.

And differing construction is exactly what dependency injection is best at.

Here's how it does it. A single interface in the shared `ui` module called `PikoPlatform` lists everything the interface needs from the platform: clipboard, system color picking, download directory selection, opening and sharing local files, full-screen dialogs, in-app updates.

Two entry points each implement it, `AndroidPikoPlatform` and `DesktopPikoPlatform`, and the entry point wires them into the root composable `PikoApp(services, platform, appearance, videoPlayer)`. Inside the UI they're reached through `LocalPikoPlatform`, with no hunting for singletons.

## Capabilities a platform lacks, expressed with null

The usual complaint about interfaces is verbosity. Take "share a file": Android has a system share sheet, desktop has no equivalent, so how do you write the interface.

Here's how it handles that. Capability fields can be nullable:

```kotlin
/** null means this platform has no in-app updates. */
val updater: AppUpdateService?

/** null means this platform offers no frame preview for downloads; the entry point is hidden. */
val videoPreview: VideoPreviewSupport?
```

Some are booleans:

```kotlin
/** System share sheet. false on desktop where there is no equivalent, so the UI hides "Share". */
val canShare: Boolean
```

Some are finer grained: whether the soft keyboard pops up, always false on desktop; and the shortcut modifier key — Ctrl on Windows, ⌘ on macOS — exposed as a `shortcutModifier` property.

The benefit of this convention is that the UI asks once whether you can do a thing, then decides for itself whether to hide the entry point. No scattering platform checks through common code for a feature that only exists on one platform. Once that kind of check piles up, common code is common in name only.

## A third tool: an intermediate source set

Interfaces handle "the two platforms really are different". But there's another category: code that is identical on both sides and only needs JDK APIs. The sample in Piko is the network proxy selector, `PikoProxySelector`, 67 lines.

It has to install a process-level `ProxySelector` into the JVM, because the SDK builds its own OkHttpClient internally, gives you no constructor parameters, and the only way in is global. That code is character-for-character identical on Android and desktop.

Follow the textbook and this is two `actual`s with copy-pasted content. What it does instead is add an intermediate source set, declaring in the build script that this group holds both JVM targets:

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

That's where the `jvmSharedMain` directory comes from. Write it once, both JVM-family targets compile it. Before writing this post I had never seen this trick in any tutorial, and of the three tools it saves the most work.

So by this point KMP has three tiers for drawing platform boundaries:

1. If it can be shared, put it in `commonMain`. That's where the vast majority of code goes.
2. If behavior genuinely differs, use an interface plus runtime injection.
3. If the code is identical and only needs JDK APIs, add an intermediate source set.

expect/actual is the fourth tier. It went unused because the first three happened to cover every need in this project. My current view is that expect/actual is for purely declarative differences — a constant, a function with no runtime state. In a genuinely complex client, fewer things land in that tier than you'd think.

![The tiers of tools for drawing platform boundaries](/images/kmp-piko/02-layers.png)

## So what do those 31,000 lines of shared code look like

`ui/src/commonMain` is the entire interface: theme, navigation, cloud drive, transfers, settings, trash, login, plus components. The entry is `PikoApp`, and each platform assembles its dependencies and passes them in.

On Android the wiring happens in `Application`:

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

On desktop the wiring happens in `main`, same shape, different implementations: `DesktopPikoPlatform(settings)`, `DesktopPikoDownloadStorage`, `FilePikoSessionStore`. Session storage is the clearest example, because the shared interface is only 21 lines:

```kotlin
interface PikoSessionStore : SessionStore {
    suspend fun loadLastAccount(): String?
    suspend fun saveLastAccount(account: String)
    suspend fun loadCredentials(account: String): PikoCredentials?
    // ...
}
```

The Android side goes through DataStore. The desktop side writes files under `~/.piko`. The UI and business code know nothing about it.

One contrast here is worth flagging. The interface says what both sides must do, but it can't govern how far apart the two implementations drift. I read the desktop implementation, and it writes session, account and password into three separate files under `~/.piko`, with the password in plaintext. On your own machine that's fine. On a shared machine, worth thinking about. Same job, same interface, and the rigor of the two sides can differ by a lot.

## Pitfalls

The architecture notes record a good number of measured pitfalls. Here are the cross-platform ones.

**The two sides' Compose versions can disagree, and the same line of code may only fail on one of them.** The desktop UI library is pinned to Compose Multiplatform 1.12.0, while Android is decided by the BOM. The reason is specific: the skiko that ships with CMP 1.13 wraps the rendering backend inside `OnScreenRedrawer`, and the playback library it uses needs to grab `Direct3DRedrawer` straight out of there for D3D11 passthrough, so opening a player window throws. In this project, upgrading CMP isn't changing one number; it means waiting for the upstream playback library to catch up.

**On Android, Compose has to go back to stable.** The version catalog says: the 1.13 alpha crashes during fast list scrolling — an interactive ListItem reads the secondary text baseline during measurement, and an aligned child looks up a parent not yet in the table. So the BOM moved to stable, and ui and foundation landed on 1.12.x.

**Changing a shared module means compiling both sides.** The author put this straight into the file meant for collaborators, because changes to `shared` and `ui` hit both entry modules at once, and compiling only one side won't show it. And the two sides' material3 versions were never the same to begin with, so the same line of code can absolutely fail on only one.

**Adding a preference means touching three places.** Once in the common interface, once in the Android DataStore implementation, once in desktop settings storage. Same for the global navigation stack: the main screen's position lives in the repository layer and is persisted, so overlays like the directory picker have to keep their own path stack — change the repository layer and the main screen's position moves with it.

**Dependencies use api rather than implementation for a reason.** `shared` uses `api(libs.cmp.runtime)` because the state types it exposes are Compose's `State` directly, and consumers can only compile if they can see those types. The counterexample is kotlinx-serialization: the SDK declares it as implementation, so it isn't transitive, and callers have to pull in their own copy.

**One licensing detail: the source is MIT, but the Android APK as a whole is GPLv3.** That's because the bundled mpv and FFmpeg are built with GPL. Fine for your own use. If you want to redistribute, read that carefully.

## What it looks like running

Piko's README has screenshots for each platform. I'm putting two here: Android on top, desktop in the same window below.

What's worth comparing isn't the content, it's the navigation. On a phone the nav bar sits at the bottom; in a desktop window it moves to the left side, and that middle decision looks only at window width, not at what device it is. So shrink the desktop window enough and it turns into the phone layout, and tablet split view goes through the same check. That adaptivity is in the shared code too.

![Piko Android](/images/kmp-piko/phone.jpg)

![Piko Desktop](/images/kmp-piko/desktop.jpg)

Functionally it does more than I expected: magnet links resolve by work, section and episode, subtitles group with their video, ad files are excluded automatically, and you can preview everything before saving and tick the files you want. Playback progress syncs with the official PikPak client, and both downloads and streaming run eight concurrent connections. It also supports lossless clip extraction, uploads, server-side extraction, whole-drive search, duplicate finding, and viewing drive usage and monthly traffic.

Worth mentioning is how it was built: commit history shows Claude as a co-author, and the README states the requirements for LLM-assisted code — understand the logic of what was added, and verify it on a real device. The author says in his post that his color sense is weak, and this interface is M3 output with manual tuning.

## Last

For a small project that needs two platforms, my advice is to carve out `shared` and `ui` first and leave them cleanly as `commonMain` only, with every platform implementation pushed into the two entry modules. That way, every time you feel like taking a shortcut and adding a platform check to a shared module, you'll pause first.

Those 67 lines of `PikoProxySelector` stuck with me the most. It looked like the most expect/actual-shaped thing in the project, and it ended up solved by adding a source set. When you have enough tools, you stop reaching for the dumbest one.

Project: https://github.com/NihilDigit/piko
