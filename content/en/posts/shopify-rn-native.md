---
title: "Shopify Went From React Native Back to Native: Is Cross-Platform Still Worth It in the AI Coding Era?"
date: 2026-09-15T15:07:44+08:00
draft: false
categories: ["WeChat"]
tags: ["Frontend"]
summary: "One number first: 12 weeks."
cover:
    image: "/images/shopify-rn-native/cover.png"
    alt: "Shopify Went From React Native Back to Native: Is Cross-Platform Still Worth It in the AI Coding Era?"
    relative: false
---

## A 12-week about-face

One number first: **12 weeks.**

Shopify's consumer app, Shop, went from proof of concept to a fully native production release in 12 weeks.

Before that, Shopify had put five years into React Native.

This is not a "the framework blew up" story. Shopify itself says it plainly: **React Native did nothing wrong, it did its job well for years.** As recently as January 2025, the company was still publicly discussing React Native's roadmap.

So why tear it up?

Shopify's director of engineering, Mustafa Ali, gave one line that I think is the thing worth chewing on:

> **"LLMs changed one of the core assumptions behind our 2020 decision."**

## Let's get the facts straight

What Shopify is doing:

- **Dropping cross-platform and going fully native** (Swift + Kotlin)
- Consumer app Shop: native rewrite done and shipped in 12 weeks
- **Next is the merchant app** — 300+ screens, deeply dependent on iOS platform capabilities. That's the genuinely hard one

What's interesting is that Shopify is not renouncing anything. They say React Native was the right call at the time, and that under the new conditions the math no longer works.

**So what changed in the conditions?**

## The core change: AI flattened the cost of building it twice

The whole case for a cross-platform framework is one sentence: **write once, run in two places.**

That value rests on an assumption: **implementing the same thing separately on iOS and Android costs too much to be worth it.**

Shopify chose React Native in 2020 on exactly that assumption.

By late 2025 they found the situation had changed:

> An agent can look at how a feature is implemented on iOS and just write the Android version — and the other way around.
>
> Engineers can also work on platforms they're not familiar with, because the agent carries most of the platform-specific work.

**See it? The cost of writing it twice on two platforms has been beaten down by AI.**

When duplicate implementation gets cheap, the value saved by sharing one codebase shrinks on its own.

And once saving headcount is no longer a crushing advantage, the benefits of native come back into view: **performance, full access to platform capabilities, no bridge layer detours, new system features on day one.**

300+ screens deeply tied to iOS platform integration — that kind of work was always a better fit for native than for a layer of abstraction.

**Native used to be avoided because it was expensive. It's not that expensive anymore.**

## Don't applaud yet: AI rewrites are far from one-click

If you think Shopify threw code at an AI and picked up a native app 12 weeks later, you've skipped the most important part of the story.

**Their first attempt failed.**

The first approach was direct: hand the existing React Native code to an LLM and have it rewrite in native.

The result — Mustafa Ali's word was **"slop"**. Adding more steps didn't fix it. Even having the model write specs and task lists before touching code produced so much code that engineers simply didn't want to maintain it.

**This is the biggest misread people make about AI coding: assuming "AI can write it" means "AI can deliver it".**

What actually got the project through was a system they built for it, called **Helix**:

- **Screen-by-screen migration**: don't try to rebuild everything at once, break each screen into smaller chunks
- **Continuous comparison**: while migrating, compare behavior against the existing app
- **Independent review**: a separate agent scans the code first, then a human signs off
- **Feedback loop**: keep the review findings so earlier problems guide later migrations

The thinking matches what Shopify CEO Tobi Lütke put on X — he said he was "considering banning Claude Code at Shopify" because engineers were submitting agent-generated code with insufficient review.

**Even Lütke, one of the most aggressive AI boosters, holds the same line: agent output has to prove itself before anyone relies on it.**

## A new bottleneck nobody predicted

There's one detail that feels very real, and everyone doing mobile should look at it:

> An agent can rewrite the code in seconds, then spend twice as long testing it in a mobile simulator.

**Writing code got faster. Verification got slower.**

This is the mismatch of the AI coding era that's easiest to miss — **your development speed went up, your verification bandwidth didn't.** However fast your hands are, the test environment doesn't speed up with them.

Shopify's fix is clever, I think: **peel the business logic out of the UI** so agents can talk to the app directly through a CLI running on the desktop.

The result: some checks that took minutes now finish in milliseconds, and the same interface can drive the simulator when needed.

**Note the fix itself — it's another proof that clear architecture beats better tools.** If your business logic and UI are fused together, no agent will save your test speed.

## Meanwhile, a wilder case

The same story mentions something else, and I think it's more interesting side by side.

Bun's creator Jarred Sumner, now at Anthropic, did a language port: **64 parallel Claude instances** moving the JavaScript runtime from Zig to Rust.

Scale: about **1 million lines of code**.

Time: **11 days**, supervised by one person.

API cost: about **$165,000**.

Before merging, the existing test suite passed on all six supported platforms.

The line in the story that lands: **a year ago this took a small team several quarters; now it's one person's 11-day sprint.**

## But some companies went the other way

Don't rush to "cross-platform is finished". This is not a one-way street.

In May this year, developer Simon Willison wrote up an encounter — he met an engineer whose company **used coding agents to merge two separate iPhone and Android apps into a single React Native app.**

Willison found it odd at the time: if agents make maintaining two codebases easy, why go to the trouble of merging?

The engineer's answer was practical: **not wanting to be locked in.** React Native did what they needed, and if things change later they can always migrate back to native.

And what Shopify is doing now is exactly that migrate-back move.

**So what actually changed is not "which direction is right". It's that migration got cheaper, and both directions became viable.**

A tech choice used to be a five-year bet, because reversing it was too expensive. Now you can flip in 12 weeks, and the choice goes from lifetime commitment to phase-by-phase decision.

Is that good for engineers, or just more tiring? I lean good — **but it demands a cleaner architecture, or you can't even migrate.**

## If you're torn, ask yourself four questions

**1. Is your pain "writing it twice" or "the two sides disagree"?**

AI can relieve the first. The second still needs architecture. If your pain is really the second, the value of cross-platform is going up — consistency is an architecture problem, and AI doesn't solve it for you.

**2. How deep is your dependence on platform capabilities?**

Like Shopify's merchant app — 300+ screens, deeply tied to iOS platform integration — native pays off for real. But if your app is mostly forms, lists and chat, the cost of shared UI is entirely acceptable.

**3. Are your business logic and UI cleanly separated?**

Shopify's "testing got slower" trap is worth everyone's attention: **if logic and UI are tangled together, AI can write as fast as it likes and verification still won't speed up.** Cut the boundary clean first, then talk about whether to use AI.

**4. Can you actually govern what the agent produces?**

Shopify only got through after building Helix. The first attempt produced slop. If you don't have a review process and verification tooling to match, "let AI rewrite it" is most likely a hole you're digging for yourself.

**The answers to these questions matter far more than which framework is more popular.**

## So back to the question: is cross-platform still worth it?

My read: **the definition of "worth it" is changing.**

**First, frameworks that live on saving headcount are under the most pressure.**

If your framework's core pitch is one codebase, two platforms, one team saved, AI is eating that pitch directly. Shopify is the live sample.

**Second, but cross-platform value was never only about saving headcount.**

There's also: consistent behavior across platforms, one release cadence, lower cognitive load for the team, business logic that doesn't need writing twice and then kept from drifting apart.

**AI made writing code cheap. It did not make guaranteeing that both sides behave the same cheap.** That's an architecture problem, not a typing speed problem.

**Third, and this is the point I most want to make: the KMP route may be sitting right on top of this shift.**

Lay out today's cross-platform options and there are really two camps:

- **Shared UI camp** (React Native, Flutter): the interface crosses too, saves the most, and gets hit hardest by "AI made native cheap"
- **Shared logic camp** (Kotlin Multiplatform): shares only business logic, each side writes its own native UI

What Shopify migrated to is **Swift + Kotlin** — note, **two native languages, and one of them is Kotlin**.

If AI really has crushed the cost of platform-specific implementation, then the camp that benefits least is shared UI. Whereas shared logic plus native UI keeps the single-point maintenance win for business logic and collects the native UI dividend.

**In other words: in the AI era, what you share matters more than how much you share.**

## My conclusion

This Shopify case gets misread in two ways:

- Misreading one: **"React Native is finished"** — no, Shopify itself denies that
- Misreading two: **"AI can do anything, it can rewrite everything"** — also no, their first attempt produced a pile of slop

**What's really happening is closer to: AI lowered implementation cost, not decision cost.**

What to share, what to keep native, how to slice the architecture, how to verify — none of those judgments got cheaper. Shopify shipping in 12 weeks wasn't because the model was that strong. It was because they built Helix to keep the model in line.

**So is cross-platform still worth it?**

Yes. But the reason is shifting from saving headcount to architectural governance.

And the answer to which framework is shifting from which one saves more code to — **which one is easier for both AI and humans to maintain well.**

Those two sentences sound similar. In practice they're far apart.

---

What do you make of Shopify's decision? Does the cross-platform versus native math work out on your project?

Comments welcome.

---

*References: [Shopify spent years on React Native — then rebuilt everything in 12 weeks (The New Stack, 2026-09-10)](https://thenewstack.io/shopify-native-ai-agents/) · [Shopify's CEO threatened to ban Claude Code (The New Stack)](https://thenewstack.io/shopify-claude-code-agentsmd/)*
