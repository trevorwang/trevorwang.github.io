---
title: "AG-UI 1.0: 31 Event Types, and a Client That Validates None of Them by Default"
date: 2026-11-18T16:00:00+08:00
draft: false
categories: ["WeChat"]
tags: ["AG-UI", "AI Agent", "Protocol"]
summary: "Five scenarios against the official client 1.0.5: three spec violations, all silently accepted. The fatal checks in the spec only fire once you wire in enforceEvents yourself."
cover:
    image: "/images/agui/cover.png"
    alt: "AG-UI 1.0: 31 Event Types, and a Client That Validates None of Them by Default"
    relative: false
---

I wired an agent up to a front end. The endpoint answered, every event arrived, the text was right, everything looked fine. Then I deliberately pushed a type that does not exist in the protocol at all into the stream. The front end said nothing and swallowed it whole.

That one move made me read the AG-UI client code from top to bottom.

## First, which layer AG-UI sits in

The agent ecosystem has three protocols now, each owning one segment. MCP handles the connection between an agent and its tools and data. A2A handles how agents split work among themselves. AG-UI handles the third segment, the stream between the agent and the user interface.

That three way split is a table the project itself lays out in its docs. I have written about MCP, and about A2UI, the one where the agent draws the interface. AG-UI was the cell I had never touched.

A2UI and AG-UI need separating first, the names are too close. The docs keep a dedicated sentence for it, and the original says: A2UI is a generative UI spec, letting an agent deliver UI components; AG-UI is the Agent and User interaction protocol, wiring an agent front end to any agent backend. One governs what the interface looks like, the other governs how data flows back and forth.

![Three protocol layers, with AG-UI filling the segment between agent and interface](/images/agui/fig1.png)

## 1.0 landed at the end of September, not today

On GitHub the repo now has 20,085 stars and 2,071 forks, is written in TypeScript, and is MIT licensed. It was created on 2025-05-07, and the last push was at 4am on 2026-11-18.

I found the official 1.0 announcement on the project blog, dated 2026-09-30.

`@ag-ui/client` has published 83 versions on npm in total, and the current latest is 1.0.5, which is what I installed. So this is not something that appeared in the last couple of days. It is a stable spec that has been running for a month and a half.

## The docs say sixteen, the SDK has thirty one

I dumped the event type enum first. The architecture page of the docs says, verbatim, that during execution an agent needs to emit any of these 16 standard event types.

But in the SDK, `EventType` comes to 31 entries after dedup.

Counting by the project's own categories: 3 lifecycle, 4 text message, 5 tool call, 2 state management, 2 activity progress, 3 sub agent, 7 in the reasoning family, 2 step, and the remaining 3 are custom, raw data and message snapshot.

The surplus is mostly those 7 reasoning events, 22.6% of all 31. The 1.0 changelog says the whole THINKING family from the 0.x era is retired and replaced by the reasoning family. The retired shapes get translated in the compatibility layer rather than simply cut off.

Sixteen is probably a leftover from the 0.x era, and that docs page never caught up. Not serious on its own, but it is the reason I decided to run it myself.

![The 31 event types counted against the official categories](/images/agui/fig2.png)

## I ran five scenarios against the official client

With `@ag-ui/client` 1.0.5 installed, I stood up a local SSE server and fed it events from the other direction. Five scenarios, results below.

Scenario one, 6 fully legal events, 6 received, text assembled into "Hello, this is AG-UI." All normal.

Scenario two, drop a `TOTALLY_UNKNOWN_EVENT` that does not exist in the protocol into the middle. The client received 6, and the unknown event turned up in the subscription callback as is, neither dropped nor warned about.

Scenario three, the first `TEXT_MESSAGE_CONTENT` chunk arrives without a `messageId`. The spec says the first chunk must carry one. The client took it, assembled the text, not a character missing.

Scenario four, the second chunk switches to a different `messageId`. In the spec that is a value conflict, and it is fatal. The client merged the two chunks into one, the former overwritten by the content of the latter, and the output became "the first chunk changed its id".

Scenario five, only `RUN_STARTED` with no `RUN_FINISHED`. The stream ended and that was that, no notice at all.

Three of the five broke the spec, and the client stopped none of them.

![Five scenarios measured on the default run() path](/images/agui/fig3.png)

## That validation is something you wire up yourself

The problem is not that the SDK never implemented it. It did, and the function sits in the export list under the name `enforceEvents`.

The problem is that by default it is not on this path. `run()` takes the SSE stream, parses it, translates it, pushes it to subscribers, and nowhere along that chain is there a mandatory validation step.

Send the same batch of events through `enforceEvents({})` and the conclusions flip immediately.

Unknown events get dropped, with a warning line alongside: the protocol describes no such event, and no middleware translated it.

A first chunk missing `messageId` throws ZodError outright, and the error points straight at `path: ["messageId"]`, expecting a string. Passing a number for `delta` instead of a string also throws ZodError, pointing at `path: ["delta"]`.

One extra field outside the spec gets stripped, with a note: removed unrecognized content on RUN_STARTED.

The legal batch of 5 all passed.

![The same batch of events with enforceEvents attached, side by side](/images/agui/fig4.png)

## Unknown and malformed get different treatment

What I most want to copy from this behavior is how it separates the two kinds of error.

Something unrecognized gets dropped, but with a warning only, no abort. The reasoning is practical: the protocol moves forward, and an old client meeting a new event is normal. Interrupting the whole stream over one unknown type would tie every client to a protocol version.

A known type with wrong content throws. There is no second reading of that. `messageId` should be a string and you sent a number, which means the sender is already broken, and carrying on only buries the error deeper.

The project wrote that line into the spec changelog as a normative clause, in BCP 14 wording.

![The two handlings, unknown events and malformed ones](/images/agui/fig5.png)

## Why the validation is off by default

I went back and thought it over, and the default is not wrong.

`run()` is for the case of "I just want to see text quickly". Plug it in and it renders, no protocol details required. The moment you really need strictness is when you write the client yourself and take someone else's agent as the input source, because then dirty data turns straight into a garbled interface.

The project left you the switch, and marked its position clearly, but it will not press it for you.

## What I do now

When wiring up AG-UI, I attach `enforceEvents` explicitly on the client side, even if it costs a few extra log lines.

The reason is simple: the agent backend can be swapped. LangChain, Mastra, Claude's hosted agents, any of them can plug in as long as what they emit is AG-UI events. Quality across those backends varies, and the front end is the only place that can apply one consistent gate. Events let through end up as strange interfaces the user sees.

One detail along the way: this SDK also implements a binary channel, HTTP plus Protobuf, frames with a 4 byte length prefix, generated from the same schema. SSE is enough for text, and it starts to matter at volume.

## Onto your own case

If your agent only talks to your own backend, with events written by your own team, the loose default path is enough, and when something breaks you can fix both sides together.

But the moment you plan to let someone else's agent plug in, or you want to keep this front end around for a third and fourth backend, that validation should be on. It is not a security measure. It moves the question of what the protocol allows out of the docs and into code.

Your agent front end today: does it parse the stream itself, or does it use this protocol? And if it uses it, is that enforce switch on?
