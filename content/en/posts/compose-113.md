---
title: "Compose 1.13 Shipped. The Interesting Part Is in 1.12"
date: 2026-10-24T16:26:16+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "Compose Multiplatform"]
summary: "1.13 has one new feature and a pile of fixes. The heavy part shipped two months earlier, when hot reload got an MCP server."
cover:
    image: "/images/compose-113/cover.png"
    alt: "Compose 1.13 Shipped. The Interesting Part Is in 1.12"
    relative: false
---

Compose Multiplatform 1.13.0-beta01 landed on October 23.

I meant to write an upgrade walkthrough. Then I read the changelog and changed my mind.

This release changes plenty, but none of it points anywhere new. The heavy thing shipped two months earlier, in 1.12.

Start with what 1.13 actually did.

## One new feature, everything else is fixes

Desktop ComposePanel in SwingGraphics mode can now pace frames to the display refresh rate. Off by default. Turn it on with a system property.

```kotlin
System.setProperty("skiko.swing.frame.pacing", "true")
```

On Linux you may also need forceTimer, which uses the system timer alongside vsync. Less precise, but sometimes necessary.

Compose embedded in Swing used to run on its own clock, out of step with the screen. That gap is now closed.

On iOS there is a new UIKitInteropSafeAreaInsetsPolicy, which controls how much of the safe area an embedded UIKit component sees. Mixed apps have wanted that switch for a while.

The only cross-platform change: AnnotatedString can now render Bullet annotations.

Everything else is a fix.

iOS fixed a crash when swiping back with the keyboard open, UIStepper misplacement in interop views overlapping safe areas, stale popup layer rendering, and a crash when selecting text right to left in an iOS app running on a Mac. Two of the four are marked prerelease fix, meaning alpha02 introduced them.

Web is mostly Safari. Text fields you cannot select on iOS 27, focus that never comes back after being lost, and the caret jumping to the end on the first tap in a multiline field. Plus a configurable font fallback URL.

Desktop got exactly one: sluggish window resizing on Windows with D3D synchronous live resize enabled.

## One thing to watch

The migration notes contain a single hard warning.

Libraries that use the experimental DragAndDropTransferAction must be recompiled, because of a binary incompatible change.

Libraries, not your app. If you are not writing drag and drop yourself, the exposure is some third-party dependency.

## A digression on version numbers

Material3 jumps from 1.12.0-alpha03 to 1.5.0-beta01, which looks like a downgrade until you notice JetBrains aligned Material3's version with Jetpack's. The components table lists it as Jetpack's 1.5.0-beta01.

Navigation goes from 2.10.0-rc01 to 2.10.0 and Navigation3 from 1.2.0-rc01 to 1.2.0, both dropping the rc. Material3 Adaptive moves from 1.3.0 to 1.4.0-alpha01.

So the real question when upgrading is not which APIs moved. It is which version numbers in your gradle files move with them.

Nobody puts that in the migration notes.

## I scanned my own three projects

The compile test did not work out. I can curl the 1.13.0-beta01 plugin straight off Maven Central, but Gradle on this machine refuses to resolve it. Aliyun mirrors and JetBrains' dev repository did not help. Twenty minutes gone, no build.

Code can still be scanned, though, and a scan tells you about risk sooner than a build does.

Three Kotlin Multiplatform projects, 492 Kotlin files between them.

NativeCanvas and NativePaint, the two typealiases whose deprecation level went to ERROR in 1.12, appear zero times.

SwingPanel and ComposePanel, zero times.

Drag and drop appears twice, both in MagnetDropTarget.kt inside Piko. It uses the standard dragAndDropTarget with AWT's Transferable to check the data type. Not the API that changed.

So at the API level these three projects have no blockers for 1.13. The only thing to touch is repository configuration, since beta builds need the dev repository added back.

That conclusion is not a great story. It does say something, though. With framework upgrades, the risk is rarely where you expect it.

## Going back to 1.12 to find the weight

1.12.0 shipped August 25. Its Highlight section has one line.

Compose Hot Reload got an MCP server, letting AI agents interact with a running Compose application in real time.

Almost nobody discussed that line. Its actual weight shows up in the tool list.

One Gradle task, hotMcpServer, brings the service up, connects to the app, waits for startup, detects shutdown, and reconnects on its own. An agent connecting in gets these tools:

status reads app and reload state including the last error. reload and await_reload trigger a hot reload and wait for it to finish. restart and reset_ui restart the app or reset its UI.

take_screenshot captures a screenshot, optionally to a path. list_windows and resize_window inspect and resize windows. get_semantic_tree reads the semantic tree including dialogs, modal bottom sheets and popups. get_ui_error and get_logs read the current error and app logs.

Then click, type_text and scroll, which simulate user input directly.

## What that loop actually closes

Put those together and an agent can do the whole thing.

Change code, trigger reload, take a screenshot, read the semantic tree to confirm the structure, read logs to confirm nothing threw. Repeat if it is wrong.

No human in the loop.

An agent walking that circle by itself is the closed loop.

The 1.3.0-beta02 release then filled in scrollable nodes. A node that can scroll now carries scrollX or scrollY with the current offset, the max offset, and a reversed flag for content scrolling against layout direction.

That looks like plumbing. For an agent it matters. Before, it knew there was a scrollable container. Now it knows where the scroll sits and whether more is left.

## Same problem as the Flutter post

Last month the Flutter team published a piece whose core claim was that code generation stopped being the bottleneck and verification is the bottleneck now.

Their fix is structural. Hot reload at 200 milliseconds, widget tests at 500, state inspection by reading the widget tree directly.

JetBrains took a different route. It does not chase speed in the framework. It opens the running app to the agent and lets the agent do the verifying.

One leans on the framework being inherently light. The other leans on opening a door.

I lean toward the second being more general, because it has nothing to do with the framework. It only asks that the runtime be reachable by a program and by an agent.

Which also means the ceiling on this route is how much that MCP server exposes. A dozen tools is enough today. More complex UI will need more.

So my question for you. Have you tried letting an agent take over hot reload and verification in your Compose project? Or are you still changing code and clicking through it yourself?

And one more. Do you think opening the runtime to agents will move slower on web and mobile than on desktop? Desktop has a JVM to hook into directly. Mobile will need a different answer.
