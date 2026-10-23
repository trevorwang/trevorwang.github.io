---
title: "Meta Muse Did Not Read Your DMs, but It Cannot Explain What It Did Read"
date: 2026-09-20T11:59:16+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "Last week Inc Magazine contributing editor Jason Aten posted some screenshots on Threads. He was chatting with Meta's new AI assistant Muse, and partway through something felt off — Muse started asking him questions about conversations in his Messages."
cover:
    image: "/images/meta-muse-trust/cover.png"
    alt: "Meta Muse Did Not Read Your DMs, but It Cannot Explain What It Did Read"
    relative: false
---

## A small incident, and Meta came out to apologize

Last week Inc Magazine contributing editor Jason Aten posted some screenshots on Threads. He was chatting with Meta's new AI assistant Muse, and partway through something felt off — Muse started asking him questions about conversations in his Messages.

The problem: Aten says he never gave Muse permission to access his messages.

So he asked back: how do you know what is in my messages?

Muse's answer went like this:

> What I see is notification previews, not your message history. I am not reading your texts.
>
> Honestly, I cannot give you the exact technical details. What I know is that the paired Mac app exposes notifications as one of its capabilities, and they reach me through device sync.

The Verge compressed that answer into three words at the time: **obviously, not great.**

But what is actually worth chewing on is what happened after. David Singleton from Meta Superintelligence Labs showed up under the post and explained the whole thing — including admitting that Muse's answer was itself wrong.

## Meta's explanation: it did not read your DMs, it misdescribed itself

Singleton's account has two parts.

The first part is permissions. He listed everything Muse needs to read messages, including full disk access for the Mac app. He stressed that all of these are opt-in, that Muse does not go around watching notifications on your Mac, and that it only syncs data from Messages after the user explicitly enables access.

The second part is the important one. Singleton says that when Muse brought up "device notifications" in its answer:

> It could not work out how to explain that feature and gave an incorrect explanation. That's on us.

Then he apologized, and said the team is improving Muse's understanding of its own internal mechanics so it can answer questions like "how do I work" more consistently and correctly in the future.

The Verge added a line at the end that reads like an old reporter's judgment: this may not be very comforting, but it makes sense — **we already knew chatbots will not tell you their secrets, they tell you what you want to hear.**

## Why this is worth more to product people than "it secretly read my DMs"

If the story were "Meta secretly read users' private messages", it would be a privacy scandal. Rage, done.

But the real situation is: **Muse did not read the DMs. Its mistake was lying about itself.**

That distinction is huge. The former is a product ethics problem. The latter is an engineering problem — and one that almost everyone building AI applications today will run into, while very few handle it seriously.

Think back to the assistant products you have built. Have users asked things like:

- Can you see my chat history?
- Where did that last sentence of yours come from?
- Do you remember what I said last time?
- How do you know that?

These questions sound like product questions. They are actually the same question: can your AI state its own boundaries clearly.

And the Muse case proves that even the most advanced products today do poorly on this one. It was not lying on purpose. It genuinely did not know how to describe itself.

## Landing it in a client: a few concrete things to do

I build an AI Chat client myself, and after reading this I wrote down a few items. Nothing complicated, but all of them are places users will ask about.

**First, the capability list has to be repeatable.**

When a user asks "what can you see", your assistant cannot improvise an answer from the model. The safe move is to make permissions and context sources structured facts, and have the assistant read them out rather than free-associate.

This is exactly the mistake Meta admitted to: Muse's answer was an explanation the model stitched together itself, not an accurate description handed to it by the system.

**Second, separate "I saw it" from "I remembered it".**

Notification previews, message history, content synced to a server — three different things, and users rank the risk of each completely differently. Muse's defense felt unsettling precisely because it mixed "notification previews" and "message history" together, and the user had no idea which one to believe.

A client should draw those levels clearly in the UI, not wait for the user to ask.

**Third, "what did I read" should be something the user can look up.**

A very plain requirement: the user can see in one screen which permissions they granted, which data the assistant read, and when it read it.

The starting point of the whole Muse incident is that the user lost that sense of control — he found out he had been "known", without knowing how.

**Fourth, when the assistant cannot answer a question about its own mechanics, it has to say "I do not know".**

Singleton's "that's on us" actually names something: rather than let the model invent a plausible-sounding explanation, let it honestly say "I am not sure about that, you can find the specifics in the settings".

That does not sound smart, but it is far better than saying something wrong.

## Back to the news itself

There is a fun contrast here.

Right while the controversy was building, Muse hit the top of the US App Store free chart, pushing ChatGPT down. The WSJ headline went ahead and labeled it: Meta's AI Agent Has a Trust Problem.

And a month earlier at the launch event, Zuckerberg's line was: personal AI assistants are Meta's biggest business opportunity.

Muse really is more than a chat toy. On the Mac it can reach local Messages, Calendar and Notes, and it can shop, book and fill forms for the user — which is exactly why it is useful, and exactly why it is unnerving.

The stronger the capability, the blurrier the boundary.

Users do not panic because you have fewer features. They panic because it knows something they do not. And the hole Muse fell into is precisely failing at knowing what it knows.

## Last thing

Personally I think this is a good reminder for anyone building AI products.

We spend enormous effort making models smarter, answers more accurate, context learned faster and deeper. But there is a corner nobody manages seriously: how to make the assistant accurately know who it is, what it can do, and what it has read.

Muse's lesson is that if you skip that corner, everything else you did well can turn into "not very comforting" overnight.

As for that Verge line, I read it several times — "chatbots will not tell you their secrets, they tell you what you want to hear".

**But people who build products have an advantage: we get to decide what they say.**

---

What do you make of this? Has a user ever asked your assistant "how do you know that", and what did it answer at the time? Comments are open.

---

*References: The Verge (coverage of the Meta Muse incident), Jason Aten's original Threads post for Inc Magazine, David Singleton of Meta Superintelligence Labs' public response, WSJ "Meta's AI Agent Has a Trust Problem".*
