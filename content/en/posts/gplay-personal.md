---
title: "I Paid $25 to Learn That Google Play Lets You Install Your Own Apps for Free"
date: 2026-10-26T09:31:22+08:00
draft: false
categories: ["WeChat"]
tags: ["Android", "Google Play", "AI 编程"]
summary: "In October, Google gave Google Play a new entrance: an app you generate by talking to AI needs no developer account, no $25, and no review — just scan a QR code. The catch is that only you can install it."
cover:
    image: "/images/gplay-personal/cover.png"
    alt: "I Paid $25 to Learn That Google Play Lets You Install Your Own Apps for Free"
    relative: false
---

I spent the better half of October on a single developer account. A $25 registration fee. Identity verification requiring a government-issued ID. The address on my payment profile had to match the ID character for character. A newly registered personal account then needs 12 testers running for 14 consecutive days. Somewhere in the middle the account got suspended and I walked the whole process again. I have written about all three of those things.

While I was running that gauntlet, on October 13, Google published an update on the Google Play product blog titled New on Google Play: More ways to create, discover, and play.

There was this line in it.

> Using Google AI Studio, you can vibe code a custom Android app. Since many of these apps start out as personal projects, through Google Play you can now launch these personal apps on your own device. This means your creations stay private, secure, and separate from the public store catalog. You only need your standard Google account to do so — no developer account required.

In plain terms: say a few sentences to an AI to generate an Android app, hit publish, scan a QR code, and the app is on your phone. No developer account, no $25, no review.

Same Google Play, two sets of gates.

![A developer account and a personal library, two sets of gates on the same Google Play](/images/gplay-personal/gplay-two-doors.png)

## What the process actually looks like

Play Console's help center opened a new page titled Personal library apps in Google Play, and it spells the steps out in detail.

Open Google AI Studio in a desktop browser and sign in with your personal Google account. Click New app, select Build an Android app, and type in what you want. Once the preview generates, click Publish in the top right corner. The first time around you connect your Google account, then click Create app.

A few minutes later, a QR code appears on screen.

Your phone is signed into Play Store with the same Google account. Scan the code, and it installs.

If you would rather not scan, the path is: in the Play Store app, tap your profile, then Manage apps & device, and find Your personal library. Everything you built yourself is in there. Tap install. Updates work the same way, go back to AI Studio, change the prompt, click Update app, and the phone updates on its own.

The line I most wanted to confirm through all of this was no developer account required. The official help page puts it even more bluntly.

> You only need a Google Account to get started.

One sentence, one account, one QR code.

![From a single prompt to installed on your phone, the five steps in the official help page](/images/gplay-personal/gplay-flow.png)

## People who already have a developer account cannot use it

This was the most surprising thing I read, and the official help page devotes a whole Q&A to it.

> Owners of Google Play Console developer accounts will need to use a different Google Account to use this feature.

Meaning if you already have a Play Console developer account and want to use this feature, you need a different Google account.

Google does not explain why. My reading is that the account systems need to stay separated. Nothing in your personal library can mix with apps you submit as a developer in Play Console, because otherwise one account could both submit public apps and bypass review to install private ones, and the boundary would blur. That is my inference, not something Google wrote.

For developers the experience is odd though. You paid $25 to publish properly, and now when you want to knock together a small tool for yourself, you have to register another account.

## The word "private" needs quotation marks

Google stresses the private part repeatedly. Not listed in the public catalog, not appearing in search results, charts, or recommendations, installable only by you.

That does not mean Google cannot see it.

The last section of the help page is explicit. When you create, modify, or publish an app in AI Studio, the tool transmits the app's package and creation metadata to Google Play, including prompts, AI-generation indicators, software dependencies and SDKs, user-facing strings, and source code.

For what? Google lists four uses: safety and security scanning, abuse prevention, policy compliance, and training and improving automated app safety classifiers.

So every prompt you write and every line of Kotlin the AI produces passes through Google's scanning pipeline. What gets waived is manual review and the publishing workflow, not data collection.

I would recommend reading the original text on this one, because "private" in Chinese tends to get read as "nobody is watching."

## Review is waived, but the rules are not

The same page lists explicit content prohibitions, and personal apps get no exemption.

No banking, loans, cryptocurrency, real-money gambling, or government services, the high-risk categories. No requesting sensitive permissions like accessibility services, SMS and call log access, binding notifications, or requesting install packages. No infringing third-party intellectual property, no impersonating existing brands, government entities, or individuals. No malware, spyware, phishing, unauthorized data collection, hate speech, sexually explicit material, or violence.

The consequence for violations is removal from AI Studio, and repeated violations terminate your access to AI Studio.

Waiving review only skips the queue and the manual check. Not one of the lines actually disappears. The difference is that Google used to stop things before they shipped, and now it deals with them after you walk into the wall.

## What kind of project it generates

This part matters if you write code.

AI Studio generates a standard Gradle project, with build scripts in Kotlin DSL via build.gradle.kts, a UI layer of Jetpack Compose with Material 3 theming, single-activity architecture with ViewModels and data classes, plus AndroidManifest, drawables, strings, and the other resource files.

Enable Firebase and you also get google-services.json, firebase_applet_config.xml, and firestore.rules. The agent adds dependencies from Maven and Google repositories on its own.

You can download it as a ZIP and import it into Android Studio to keep going.

The limitations are stated just as plainly, four of them.

Client side only, no server component, so anything involving secret management, multiplayer, or Google Workspace APIs is out. Single activity and single module, so structures beyond that scale do not work. Kotlin and Jetpack Compose only, Java and XML layouts are unsupported. No NDK, so C and C++ cannot come in. Wearables and Android TV are unsupported too, phones and tablets only.

One detail I found interesting: the emulator in the browser runs entirely in the cloud, so you do not need the Android SDK, Android Studio, or a local emulator installed. But it does not support camera, NFC, or Bluetooth, GPS is simulated, and there is no Google Play services, so Google Sign-In and Maps do not work in the emulator and need a real device to behave.

The ceiling on this kind of project is clear. Fine for personal tools, small apps, prototypes. Not for stacking a large codebase on top.

![The Android project structure AI Studio generates, and the four hard limits Google lists](/images/gplay-personal/gplay-limits.png)

## One button, two exits

For me the most interesting detail in all of this turned out to be the Publish button in AI Studio.

It has two exits. One leads to the personal library, zero gates, installable only by you. The other leads to the internal testing track in Play Console, which needs a $25 developer account and a complete developer profile, and reaches up to 100 testers.

Same screen, same generation run, and the rules differ that much.

Google also lays out the path forward in the blog post. In the coming months you will be able to share personal apps with a small circle of friends and family, and at that point the app goes through a Trust & Safety review first. If you eventually want the whole world to use it, you can upgrade to a full developer account, and every app in your personal library migrates to Play Console as a draft on the internal testing track.

My sense reading that section is that Google has split "build a little tool for myself" and "build a product for other people" into two entirely different entrances, with an upgrade path between them.

Previously only the latter existed. If you wanted to put something AI had just generated onto your own phone, you first had to become a developer. Now the former has its own door, and the price of entry is admitting you are not a developer.

![The three steps Google has laid out: personal library, sharing with friends and family, upgrading to a developer account](/images/gplay-personal/gplay-stairs.png)

## This feature is not available in China yet

It is in beta, open to users 18 and over in the US, rolling out over the coming weeks. Play Store app version 53.5 or later. Other countries get "near future," with no timeline from Google.

I am not in a supported region, so I never touched the feature. Every process, limit, and quote above comes from three official documents: the October 13 Play product blog, the Play Console help page Personal library apps in Google Play, and the AI Studio Android build documentation. Which sentences are official text and which are my inference is marked in the body.

So here is the question.

If a friend in the US can say a few sentences to an AI and have an app installed, while you are still grinding through 12 testers for 14 straight days, would you register a plain Google account to route around it? Or was that $25 and those five gates always buying the right to let other people install your app?

Think one layer down. What Google knocked down this time is only the half of the wall labeled "for my own use." The other half has not moved. When enough vibe coders want to share with friends, does that Trust & Safety review become the new five gates?

Do you have one of those small tools that only you would ever use, but you never bothered to open an account for it? If a single prompt could put it on your phone right now, what would you build first?
