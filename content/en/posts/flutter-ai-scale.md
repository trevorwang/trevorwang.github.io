---
title: "AI Can Write Swift and Kotlin Now. So What's the Point of Cross-Platform?"
date: 2026-10-23T14:31:26+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "Flutter"]
summary: "On September 10, Shopify's head of mobile published a post."
cover:
    image: "/images/flutter-ai-scale/cover.png"
    alt: "AI Can Write Swift and Kotlin Now. So What's the Point of Cross-Platform?"
    relative: false
---

On September 10, Shopify's head of mobile, Mustafa Ali, published a post.

The title was blunt: native is now the future of mobile at Shopify.

Six years ago they went all-in on React Native. Last January, Ali wrote a retrospective saying RN had a bright future. Now they are reversing course, and the stated reason is one thing: coding agents have crushed the cost of writing everything twice.

The switch worked out. Six engineers took Shop App from proof of concept to the app store in 12 weeks. iOS cold start improved 23%, Android 50%.

![Migration diagram from Shopify's engineering blog](/images/flutter-ai-scale/shopify-native.png)

But the most valuable line in that post is not a number.

Ali writes that an agent can rewrite the code in seconds, then spend minutes working out whether it actually works. Simulator control is too slow, so getting state means going through the accessibility tree or taking screenshots.

His words: if the model cannot quickly verify its own work, it does not matter how good the model is.

That complaint made me read two posts side by side. The Shopify piece from September 10 and the Flutter team's response forty days later are arguing about the same thing.

## The bottleneck is not generation, it is verification

The Flutter post went up on October 20, by Carlos Mendonça and Emma Twersky. Its title is just as blunt: AI makes it easy to migrate, Flutter makes it easy to scale. It exists to answer the question Shopify threw down.

The whole piece rests on one claim. Code generation stopped being the bottleneck a while ago. Verification is the bottleneck now.

Two numbers back it up. 73% of developers say AI makes them more productive when starting a task. 46% do not trust AI's debugging accuracy without a human checking it.

Squeeze from both sides and the blocker sits on the human side.

## The bill for three steps

The team published a comparison table putting native and Flutter next to each other across the three verification steps.

![The same table rearranged in Chinese, values and wording unchanged](/images/flutter-ai-scale/verify-table.png)

Code-to-running latency: native takes 2 to 5 minutes per cycle. Build, link, archive, install on the simulator. Four debug rounds burn twenty-odd minutes.

Flutter stays under 200 milliseconds. Hot reload injects the change into a running Dart VM and keeps state.

Automated tests: native walks the full simulator lifecycle and a background emulator. Flutter's headless widget tests run in an isolated environment in under 500 milliseconds.

The UI and state-check row is where it gets interesting.

Native screenshots the simulator and hands the image to a vision model to guess at the layout. Probabilistic, and slow. Flutter goes through MCP to query the Dart VM Service and DevTools, reading the widget and render object trees directly. Deterministic, structured.

Chain the three and you have a complete loop. Every step queues on the native side. On Flutter a round trip is milliseconds.

## Harness beats prompt

Fast verification is one thing. Getting the agent to actually use it is another. The team ran an internal experiment called Project One Shot, letting an agent fix a production bug in camera_android_camerax on its own.

The plugin manages Android CameraX lifecycles and surface texture binding, the hairiest kind of Flutter code there is.

Given vague instructions like watch out for memory leaks, the agent failed every time.

It only worked with a deterministic harness. AST analysis over MCP, package-level domain skills, enforced test-driven guardrails. Reproduce with a failing test, write the smallest fix, finish with the full test suite and leak checks.

No room for the agent to freestyle. Every step has its acceptance criteria nailed down.

Three things shipped alongside. An official Dart and Flutter agent plugin bundles an MCP server with ten Flutter and fourteen Dart skills, so the agent can reach the Dart Analyzer and Widget Inspector directly.

A dedicated agent inside Antigravity audits accessibility issues and submits fixes that match existing conventions. Plus an Agent Skills CLI built with Serverpod: one dart run skills@get installs the skills a package ships with into your agent.

The benchmark numbers: Gemini 3.5 Flash from 84.1% to 85.5%, Gemini 3.1 Pro from 89.6% to 92.7%. Modest gains, run by the team on its own eval suite.

## Where this actually lands

The post lists a set of deployment targets well beyond phones. Toyota and Lexus head units, the Xiaomi SU7 center console, Nubank's fintech app with a hundred million users, Universal Music.

The post does not push this angle. I think it is the most convincing part.

What these share is absurdly expensive verification.

Flashing a head unit takes tens of minutes. One production incident in a fintech app costs real money. Music app users tolerate zero jank.

In places like these, a minute off the verification loop is money and time you keep.

Look further out. Once agents writing code is normal, one codebase across platforms saves more than headcount. It saves time.

An agent produces a change, hot reload shows it on phone, desktop and web at once. That is an order of magnitude faster than compiling, packaging and installing per platform.

That is also why the Flutter team has been pushing into automotive, desktop and embedded.

The second half of cross-platform may not happen on phones.

## A few things I do not buy

The app store rating section does not hold up. It cites Apptopia's study of 201,000 apps. On Google Play, Flutter apps average 4.1 stars, and so do non-Flutter apps. On the App Store both sit at 4.0. The conclusion offered is that users do not care about the framework.

All that shows is that cross-platform did not hurt ratings. Flip it around and it also shows cross-platform did not help. Using a tie to argue for adoption is missing a step.

That 200 milliseconds is the time to inject a change. It excludes writing the code, thinking it through, switching windows.

It cannot change class structure either. Add a field or restructure the state tree and you need a restart.

The table measures code-to-running latency, which is an honest definition. Readers will take it as development being twenty times faster, and those are very different claims.

The enterprise examples have survivorship bias too.

Xiaomi SU7, Universal cutting 45% of its code, Toyota and Lexus, Nubank's hundred million users. All real.

No post ever covers the company that moved to cross-platform for two years and then tore it out. Arguments from these examples are half evidence.

![The eight arguments the team published, enough to argue with in a team meeting](/images/flutter-ai-scale/talking-points.png)

## How to do the cross-platform math

Line up both sides first. Shopify's call: the cost of writing twice has been crushed, and what is left is not worth giving up native capabilities for. Flutter's call: coordination costs remain.

The example in the post is specific. Same checkout state machine, two instances of the same model. One writes Swift and drops stale events through an async stream. One writes Kotlin and queues events through a reactive flow.

Both pass local unit tests on day one. Weeks later Android checkout conversion drops three points, because under network jitter the retry timing differs.

![From the original post: parallel agents do not remove coordination cost, they introduce nondeterministic drift](/images/flutter-ai-scale/parallel-agent-challenge.png)

Both positions hold. They are doing different arithmetic.

I also do not think this pivot is evidence that cross-platform is dead.

This is a company with hundreds of screens, over a thousand engineers, and mobile as a core investment. At that scale, rebuilding for 23% faster cold start and a better crash rate pencils out.

For five people maintaining a mid-sized app, the same move probably does not.

What decides whether you go cross-platform is team size, how many years you will maintain it, and how long your verification chain is. AI changes none of those.

Here is a question worth arguing about instead. If agents really do get a verification cycle down to a few hundred milliseconds, how much longer does native-feels-better hold up.

I would like to hear what is actually on your desk. Is your project single-platform, dual, or cross? What drove that call: performance, headcount, or a boss who said we are going native?

One more question: do you think Compose Multiplatform has a shot? It is cross-platform too, arrived years after Flutter, but it stands on half the Kotlin and Android ecosystem.

Drop a comment. I will pick a few representative ones and write them up as a follow-up.

Both source posts are here. Twenty minutes total.

[Native is now the future of mobile at Shopify](https://shopify.engineering/back-to-native)

[AI makes it easy to migrate. Flutter makes it easy to scale.](https://flutter.dev/blog/ai-makes-it-easy-to-migrate-flutter-makes-it-easy-to-scale)
