---
title: "ChatGPT Has Ads Now: Click One and You Don't Land on a Website, You Land on a Brand AI"
date: 2026-09-19T07:15:54+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "One detail first, and it may strike you as a little strange."
cover:
    image: "/images/chatgpt-ads/cover.png"
    alt: "ChatGPT Has Ads Now: Click One and You Don't Land on a Website, You Land on a Brand AI"
    relative: false
---

## One counterintuitive detail

One detail first, and it may strike you as a little strange.

When you click an ad in ChatGPT, **it does not open the brand's website.**

It **starts a conversation** — with an AI agent sponsored by the brand.

The feature is called **Sponsored Agents**. OpenAI announced it on its own blog on September 16 and is testing it in the US with a set of advertisers.

The official description:

> Sponsored Agents give users the option to go deeper. After seeing a relevant ad, users can choose to start a **clearly labeled** conversation in ChatGPT with an agent sponsored by the brand. They can say what they care about, ask follow-up questions, and when they are ready for the next step, click through to the brand's site.

**Watch the wording here. It carries a lot:**

- **"Clearly labeled"** — OpenAI knows the sensitive part is whether users can tell an ad from their assistant talking
- **"the conversation is kept separate from your original conversation, and separate from ChatGPT's independent answers"** — an ad conversation must not pollute your existing context
- **"when ready for the next step, click through to the site"** — the link is still there, but it became **an option after the conversation**, not the destination of the ad

**This is a substantive change in ad format.**

The old ad logic was **send you away**: click, redirect, leave. Every platform is anxious about click-through rate, because a click means the user left.

Sponsored Agents invert it: **keep you here and talk.**

## This is not a side experiment. It is version one of AI-native advertising

Digiday quoted **OpenAI CFO Sarah Friar**, and I think that quote is the key to the whole thing.

At Goldman Sachs' Communacopia tech conference on September 8, she said the ads currently sitting under ChatGPT answers are **just a "basic starting point."**

Then this:

> **This is all before we've really launched a format that feels truly endemic to AI.**

(This is all before we've really launched a format that feels truly endemic to AI.)

Translation: **what you are looking at is not even the real version yet.**

**What does AI-native mean?**

Traditional internet advertising is **designed for reading web pages** — banners, feeds, auction-ranked results, all built so you bump into them while **scanning**.

AI-native ads are designed for **an assistant in conversation with a person**. In a conversation, the user is **asking questions**. So the ad should not be a banner. It should be **something you can ask questions of**.

**The ad goes from display item to salesperson.**

One brand is already testing it: **Wayfair**. Their paid media director said something refreshingly plain:

> Wayfair is taking part in OpenAI's Sponsored Agents pilot, exploring how **conversational commerce** can help customers make smarter, more confident decisions.
> We are testing at limited scale, with **guardrails around product accuracy, transparency and service handoff.**

The word **guardrails** matters. Brands know that having an AI speak for them is a completely different order of risk from buying a banner.

## One number, and one line that got overturned

There is something almost theatrical in the details here.

The Wall Street Journal reported in early September that **ChatGPT ads have hit a $1 billion annualized revenue run rate.**

The same piece notes that **Sam Altman once called ads a last resort.**

"Ads are a last resort for our business model." "I hate ads." He said both. And now OpenAI is testing Sponsored Agents, letting advertisers buy directly from **HubSpot and Shopify** dashboards, and has brought in **Amazon Ads** for a conversational ad pilot.

36Kr's headline nailed it: **"$1 billion is just the starting point."**

**From "I hate ads" to "$1 billion annualized and still scaling" is roughly the distance of a compute bill.**

## Meanwhile, another company took a completely different road

In the same week OpenAI was laying down an ad network, **Anthropic**'s news read differently:

- **Profitable two quarters running** (multiple outlets, the 14th)
- Annualized revenue of **$65 billion** in July, projected to **top $100 billion** this year (sources, the 18th)
- IPO in motion, **possibly on Nasdaq as early as October**

One more item from the same window is more telling: **21st Century Business Herald ran "Anthropic races to Nasdaq, OpenAI is not coming."**

**The two most similar companies publicly split on business model this month:**

**One trades attention for money** — ChatGPT has 800 million users, and turning attention into ad revenue is the road Google and Meta have walked for twenty years.

**One trades capability for money** — Claude runs on subscriptions and enterprise API, sold to people willing to pay for a stronger model.

What is interesting is that **both work**:

- OpenAI's user base is too large for ads not to be the obvious choice
- Anthropic turned "we don't do ads" into a selling point — for enterprise buyers, a model **that is not bent by ad incentives** is valuable on its own

**"No ads" went from a technical choice to a market position.**

## So what does this mean for people building AI products

I am writing an AI chat client myself, so my angle on this is a bit different.

**First, conversational ads will redefine what counts as good product design.**

In the past, client design meant figuring out how to show an ad without getting in the user's way.

Sponsored Agents make **the ad a participant in the session**. That creates new problems the product has to handle:

- **How do you make it unmistakable that I am now talking to a brand, not my assistant** (OpenAI uses labeling plus conversation separation)
- **How do you hand the user from my assistant to the brand agent, and then hand them back, without it feeling clumsy**
- **What is the right way to isolate context**

**None of these have answers in current product design practice.**

**Second, not selling ads becomes a durable differentiator.**

Anthropic's road shows one thing: **when your competitor starts selling ads, your refusal becomes pricing power.**

The same logic applies to anyone building a vertical AI product — **put ads in your product and users will reprice how much they trust what it tells them.**

**Third, and most practical: advertisers will start demanding AI-readable assets.**

When an ad goes from banner to agent, brands need **product knowledge built for an AI** — not campaign assets, but a knowledge base that can be interrogated, answer details, and say "I am not sure about that" when it isn't.

**That may be the next real gap in demand.**

## One hard engineering problem: keeping the ad in bounds

Sponsored Agents sound like a product design question. In implementation it is all engineering.

**Problem one: isolating context.**

OpenAI specifically stresses that the sponsored agent's conversation is kept separate from your original one. That sentence hides a hard architectural constraint:

**The brand agent must not be able to read what you told ChatGPT earlier.**

Think about it. A user mentioned three hours ago that they are paying off a mortgage and money is tight, then an ad for a sofa shows up. **If the brand agent can read that, it is not advertising anymore, it is harvesting.**

So there has to be **a session-level isolation boundary**, and it has to be verifiable and auditable.

**Problem two: making the label actually work.**

"Clearly labeled" sounds simple, but product history has almost no successes here — **user attention to ad labels is basically zero.**

So what works is not a label but **changing the whole interaction shape**: a different avatar, a different bubble color, plain text saying this is the brand's AI, not ChatGPT. The goal is that users **cannot confuse the two visually or interactively.**

**Problem three: handoff without accidents.**

The user finishes with the brand agent and moves to the next step: view the product, place the order. If the agent got a parameter wrong — **who is liable?**

That Wayfair line about **guardrails around product accuracy, transparency and service handoff** is exactly this. Brands have realized: **let an AI speak for you and every sentence it says has legal consequences.**

**None of these three has a standard answer yet. Whoever works it out first defines the engineering spec for the next generation of AI ads.**

## But let's stay sober for a minute

**First, the results are nowhere near proven.**

The very title of that 36Kr piece — A Complete Guide to Why ChatGPT Ads Underperform and How to Optimize Them — tells you the story: **ads underperforming is already an industry consensus that needs a dedicated guide.**

Advertisers care about conversion and ROI. Turning "chatted with the brand AI for three turns and left" into money is, for a CFO, still an unsolved sum.

**Second, trust is the most dangerous thing here.**

ChatGPT's product strength rests on the default assumption that **it is on your side.**

Once ads enter the conversation surface, that assumption erodes. OpenAI clearly knows — hence the repeated emphasis on clearly labeled and kept separate. **You can see them walking carefully along that line.**

**Third, even OpenAI is putting limits on the ads.**

A detail: on September 10 word came that **OpenAI bans ads for competing AI products in ChatGPT** (Adobe and others were affected).

**It sells the inventory and screens who may buy it.** What that tells you: **the rules of this ad slot are not set by the market, they are set by OpenAI.**

## Last

Lined up together, these are worth a look:

- **OpenAI**: $1 billion annualized in ads, Sponsored Agents live, from I hate ads to building an ad empire
- **Anthropic**: two profitable quarters, $100 billion annualized in sight, IPO in October, no ads as a selling point
- **Amazon, Shopify, HubSpot**: racing to plug into the ChatGPT ad pipeline
- **Wayfair**: testing carefully, guardrails first

**Together they say one thing: the free lunch of the AI assistant is being priced.**

Only this time the ad does not look like an ad — **it is not something stuck in front of your eyes, it is the character sitting across from you, ready to answer your question.**

---

I want to ask you:

**If ChatGPT popped up a "chat with the brand AI" button, would you click it?**

**Or put another way: what would an acceptable ad in an AI assistant look like to you?** Comments are open.

---

*Sources: OpenAI's blog (9/16), PYMNTS, Digiday, The Wall Street Journal, Reuters, 36Kr, Cailian Press and other public reporting.*
