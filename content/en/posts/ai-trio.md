---
title: "Are the Big Three Back in the Game?"
date: 2026-10-01T07:46:19+08:00
draft: false
categories: ["WeChat"]
tags: ["Tech"]
summary: "First reaction: \"oh, finally.\""
cover:
    image: "/images/ai-trio/cover.png"
    alt: "Are the Big Three Back in the Game?"
    relative: false
---

## Saw Gemini 4 this morning

First reaction: "oh, finally."

Second reaction: "I can't use it anyway."

On September 30, Google's blog published Gemini 4 Argon, bylined to Koray Kavukcuoglu, senior VP at Google DeepMind. It hit the Hacker News front page the same day. The opening line is restrained:

> Today, we're announcing our new frontier model, Gemini 4 Argon, which is rolling out to a set of trusted cyber defenders through our Fairwind Program.

Which translates to: it goes to a batch of trusted cyber defenders first. Not to you.

I know this road well. This is how model vendors have shipped frontier capability for years. First the US government's voluntary pre-deployment evaluation, then a small circle of trusted testers, and only then developers, enterprises and ordinary users. The post even spells out the order: "starting with paid API customers and Google AI Ultra subscribers".

So the accurate statement is not "Gemini 4 shipped". It is "Gemini 4 was announced, and a small batch went out."

## Right now you really can't use it

I didn't want to get anyone's hopes up, so I checked two places myself.

First, Google's own developer docs. On the Gemini API model list page (marked updated September 24), the newest entries are still Gemini 3.8 Flash, 3.8 Live, 3.8 Flash TTS. No argon endpoint. Which means even with an API key you can't call it.

Second, prediction markets. Polymarket runs a "when will Gemini 4.0 launch" market with $880K in volume and rules written in stone:

> For this market to resolve to "Yes," Gemini 4.0 must be launched and publicly accessible... A closed beta or any form of private access will not suffice.

A closed beta doesn't count. There has to be a publicly reachable entry point. So on that market, September 30 sits at 1%, October 15 at 68%, October 31 at 88%, November 30 at 94%. Bettors putting real money down are saying they don't think that day counted as a launch.

Put the two together and the weight of this gets clear. The cards are face up, but they're still in Google's hand.

## What Argon actually puts on the table

Can't use it, but the post carries a lot. Here's what I pulled out.

Output cap raised to 1M tokens. The post says from 64K to 1M, and calls it "industry-leading". This matters more than it sounds: a model that can write hundreds of thousands of tokens in one go turns "long-horizon task done in one shot" from a demo into something you can configure.

A whole row of benchmarks. 77.9% on DeepSWE v1.1, which the post calls the new SOTA on real-world, long-horizon software engineering tasks. 51.3% on AutomationBench, first place in Zapier's end-to-end business workflow eval. 91.7% on LVBench for long-video understanding. 68% on CWE-bench v1 for vulnerability repair, tied for first.

It also name-checks Vals Index, Vals Finance Agent v2 and Harvey's legal agent benchmark, all framed as leading.

Pricing pinned right up against competitors. Introductory $2 per million input tokens, $10 output, 95% off cached input, back up to $4 and $20 after the intro window.

The most concrete part is how Google itself uses it. Three internal cases, and I find them more meaningful than the benchmarks: the quantum computing team used it to optimize subroutines, beating the published baseline by 40%; a set of agents analyzed fleet-wide performance data for memory optimization, freeing 300 TiB up front with 500 TiB to 1 PiB expected at the end.

Then large-scale C/C++ to Rust migration, from re2 and libgav1 all the way to Fuchsia's Zircon kernel, over 800,000 lines. The libgav1 case is specific: the agent converted 32,000 lines of SIMD code into safe Rust that runs 2.7x faster than the existing Rust port.

Security it lays out fairly openly. The build for trusted defenders ships without cyber guardrails so they get the full attack surface. The public build emphasizes protection against indirect prompt injection (claimed leading on Gray Swan's IPI benchmark), plus mismatch monitoring over chain of thought and actions. Wiz already used it to find a high-severity vulnerability in medical software used by hospitals worldwide, one that a previous frontier model missed.

## What everyone else was doing the same day

This is where the "big three" line comes from. This half month was not a solo Google show.

Timeline first. Anthropic shipped Fable 5.1 and Mythos 5.1 on September 1, then Opus 5.5 on September 22 — one line summary: "matches Fable 5.1, 40% cheaper than Opus 5". Then Sonnet 5.5 on September 28, "30% faster, up to 30% cheaper on most work".

On OpenAI's side, September 30 (same day) brought GPT-6.1 Sol, with a subtitle blunter than Google's:

> Near-Astra intelligence for a fifth of the price

Near-Astra intelligence at a fifth of the price. $2 per million input, $10 output, $0.10 cached input. A few concrete numbers: matches Astra on DeepSWE v1.1 at a fifth of the cost; beats Opus 5.5 on GDP.pdf at less than half the cost per task.

On Terminal-Bench Science, the scientific workflow benchmark, it costs $5.47 per task. Opus 5.5 is $23.21, Astra is $23.80.

You may have noticed something already. Google and OpenAI both report DeepSWE v1.1 and AutomationBench, and both landed cached input at $0.10. Benchmark framing and pricing framing converging at the same time means both are staring at the same customers.

Now look at what each of them has been filling in. Since Gemini 3 late last year, Google has shipped 3.5, 3.6, 3.7, 3.8 Flash variants along the way, the planned 3.5 Pro vanished without a trace (that's media reporting, Google never said), and Argon only comes out now. Anthropic's updates all landed on cheaper and faster. OpenAI's Sol is Astra's capability sold at a discount.

## So does "the old guard is back" hold up

My read: rather than back, they changed how they play. Three things are pretty obvious.

One, they stopped competing on chat and started competing on whose work gets done. This month the keywords across all three are long-horizon software engineering, financial research, legal drafting, vulnerability repair, business workflows. All expensive, specialized, budgeted places.

Two, frontier models now ship in limited release. Google even says plainly this time that guardrails get hardened as access widens, and public release is "gradual". That's not the era of publish a blog post and everyone can call it. Frontier capability looks more and more like a controlled resource.

Three, the price war changed its unit. It used to be dollars per million tokens. Now both sides quote cost per task. Sol's "$5.47 per task versus $23.21" is the most effective ad copy going right now.

So I'm not comfortable with the word "back". Nobody got chased out and returned. What happened is that once open source took cheap and self-hostable, this batch of old guard turned around to take expensive work and distribution.

For people writing AI clients, two parameters are worth actually remembering. One is cached input pushed down to $0.10 per million tokens, which is a direct cost variable for clients with long sessions and repeated context. The other is that the eval names changed: DeepSWE v1.1, AutomationBench, CWE-bench are closer to "can you actually do the work" than the MMLU generation was. Next time you pick a model, these beat the overall leaderboard.

## One honest boundary

There is no API key from any of them on this machine (I checked, all empty), so every number in this post — the 77.9%, the 51.3%, all the pricing — comes from official launch pages and public benchmark leaderboards. I have not run Argon, and can't. It is only open to trusted defenders in the Fairwind program.

What's official, what's media reporting, what's third-party prediction, I marked in the text. For example "3.5 Pro was cancelled" is media reporting. Polymarket odds are market prediction, not fact.

As for my own next step: the day it opens to paid API customers, I'll run it against the AI Chat client I have and watch two things — whether a one-million-token output cap is actually usable in a real conversation, and how much that $0.10 cached input saves over long sessions. Then I'll write one with data.

Do you think this round is "the old guard is back" or "the old guard changed how it sells"? When did you last genuinely pay for someone's API?

References:
- Introducing Gemini 4 Argon (Google official): https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-argon/
- Introducing GPT-6.1 Sol (OpenAI official): https://openai.com/index/introducing-gpt-6-1-sol/
- Anthropic Newsroom (Opus 5.5 / Sonnet 5.5 / Fable 5.1): https://www.anthropic.com/news
- Gemini API model list: https://ai.google.dev/gemini-api/docs/models
- Polymarket "Gemini 4.0 released" market: https://polymarket.com/zh/event/gemini-4pt0-released-by-june-30-2026
