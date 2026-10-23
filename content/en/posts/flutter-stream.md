---
title: "Streaming Chat in Flutter: What Three Pitfalls Actually Cost"
date: 2026-10-17T10:34:47+08:00
draft: false
categories: ["WeChat"]
tags: ["Flutter"]
summary: "Anyone writing an AI chat client runs into this: the model does not hand you the answer in one go. It dribbles out token by token, wedging tool calls, code blocks and stretches of thinking in between. The user staring at the screen is really watching a tree grow while someone keeps rearranging it."
cover:
    image: "/images/flutter-stream/cover.png"
    alt: "Streaming Chat in Flutter: What Three Pitfalls Actually Cost"
    relative: false
---

## One reply streams for twenty minutes, and the UI has to grow with it

Anyone writing an AI chat client runs into this: the model does not hand you the answer in one go. It dribbles out token by token, wedging tool calls, code blocks and stretches of thinking in between. The user staring at the screen is really watching a tree grow while someone keeps rearranging it.

The GetPullRequest team wrote a long post recently about how they built this in Flutter. The sharpest line is about Flutter's official GenUI package:

> It only drew the UI after the model's full response had arrived and been parsed.

Which means it waits for the whole answer to arrive and be parsed before drawing anything. For a normal Q&A, waiting a few seconds is fine. For an agent doing twenty minutes of work, it is not: the tool card should have appeared long ago and you are still looking at a spinner.

I reproduced the key scenarios from that post on macOS and measured them. Three pitfalls here, each with a number you can produce yourself.

## Pitfall one: re-parsing Markdown on every token

The most naive approach: a slice of text arrives, you throw the accumulated full text at a Markdown parser, get the node tree, render it. Looks like the obvious thing to do. The cost is that you re-parse what has already been parsed, every single time.

I took an 886-character agent reply as a sample: prose, a three-row table, a dart code block, a two-level nested list. I sliced it into 222 four-character chunks to simulate streaming. Here is how the three approaches compare.

| Approach | Total time | Notes |
|---|---|---|
| Full re-parse on every chunk | 28 ms | 222 parses, slowest single parse 0.62 ms |
| Re-parse only inside the last, still-growing block | 4 ms | Completed blocks parsed once |
| Parse once after everything arrives | 0.148 ms | The cost of the GenUI approach |

28 milliseconds does not sound like much, but it is not spread evenly across 222 chunks. The first 50 chunks average 0.046 ms, the last 50 average 0.199 ms. It gets more expensive as you go, because the accumulated text gets longer. This is quadratic work: for an 886-character reply, the naive approach processes about 98K characters, 111 times the original.

Real-length answers make it obvious. Same approach, reply length doubles, time goes up two to three times:

```
886 chars / 222 chunks: 16 ms
1774 chars / 444 chunks: 61 ms
3550 chars / 888 chunks: 229 ms
7102 chars / 1776 chunks: 899 ms
```

A frame budget is 16.7 ms. By the 3550-character tier, just parsing the tail eats several milliseconds, and add layout and paint on top and dropped frames are inevitable.

![Three parsing strategies for the same reply](/images/flutter-stream/fig1.png)

The post's fix is block-level incrementality: completed blocks parse once, only the growing one is re-parsed, and "notify the UI on every chunk" becomes "coalesce notifications into one microtask."

## Line-by-line parsing shreds the structure

There is a sneakier pitfall, tied to a product requirement: letting users select and comment on individual lines. If your render layer is one widget per line and each line is parsed separately, Markdown structure falls apart.

The same content, parsed line by line, produces this tag sequence:

```
[p, p, p, p, p, pre, p, p, pre, ul, li, pre, pre]
```

Parsed in chunks it produces this:

```
[p, table, thead, tr, th, th, tbody, tr, td, td, tr, td, td, pre, ul, li, li, li]
```

Compare the counts. The table: 0 tables when parsed line by line (a four-row table turns into five lines of pipe-separated text), 1 when parsed in chunks. The code block: 4 pre tags line by line (it should be one unit), 1 in chunks. The nested list: only 1 li line by line (it should be three levels), 3 in chunks.

The nested list case is the easiest to miss. In Markdown, four leading spaces mean "this is code," so the moment the second level of a list gets parsed on its own it turns into a code block, shown on screen as a grey box with a bare minus sign in front of it.

![Structural difference between line-by-line and chunk parsing](/images/flutter-stream/fig2.png)

One more basic pitfall while we are here: the markdown package only parses tables with the extension explicitly enabled. The default `Document()` yields 0 tables; switch to `ExtensionSet.gitHubWeb` and you get 1.

And one that is specific to streaming: a chunk can break in the middle of three backticks. When I fed that half-finished text to the parser, the one or two trailing backticks became literal characters in the body, so two lonely `` ` `` pop up on screen and only turn into a proper code block when the next chunk arrives. And if it happens to be a complete triple backtick with no content yet, you immediately get an empty code box.

## Pitfall two: you scroll up, it yanks you back to the bottom

The chat list is reversed (newest at the bottom, offset 0 is the bottom). The user scrolls up 300 pixels to reread history. A new token arrives, and if the code does no checking and just scrolls to the bottom, the view is dragged back 300 pixels by force.

Measured: offset jumps from 300 to 0, the viewport is pulled 300 pixels. The paragraph being read vanishes.

Change it to "only auto-scroll when within 200 pixels of the bottom" and at that same offset of 300, the offset stays at 300 when new content arrives. The user is left alone.

The second source of jitter is card expansion. A card above the viewport grows from 100 pixels to 400, visible content shifts 300 pixels, and whatever was under your finger drifts away. The fix is to measure the height difference first, then after that frame's layout, move the scroll position back by the same distance. Measured: compensating 300 pixels leaves 0.0 pixels of residual anchor drift.

The post also mentions the reverse case: expanding is easy, collapsing is hard, because Flutter has already adjusted the scroll position before you get a chance to fix it. You have to remember the position from before the change.

## Pitfall three: treating one long reply as one widget

The first two pitfalls live at the text layer. The third is at the list layer. The old way stuffs one reply into a single Column with tens or hundreds of blocks inside. The problem is that as soon as Flutter sees any part of that widget on screen, it has to build the entire Column.

Measured with 500 blocks:

| Rendering | First render | Blocks actually built | One update |
|---|---|---|---|
| Whole reply in one Column | 232 ms | 500 | 30 ms, rebuilds 500 |
| One block per row with ListView.builder | 24 ms | 14 (only what is visible) | Rebuilds only the changed row |

232 ms is fourteen frames. It janks the first time you open it. A 30 ms update also drops two frames. Split into blocks, the first render builds only 14 and is an order of magnitude faster.

![Two ways to render 500 blocks](/images/flutter-stream/fig3.png)

Then there is notification frequency. Twenty chunks arrive in one event loop. Call `notifyListeners()` on each and the UI is asked to refresh 20 times. Coalesce them into one microtask with a small flag and you get 1 notification. Same result on screen, nineteen copies of wasted work gone.

The line from that post I agree with most, and the conclusion of all three sections:

> the widget that owns a stream should be the smallest widget that shows it.

Which means: whoever owns the stream owns the listening. Do not let one big thing listen on your behalf.

## How to read these numbers

The caveat first: these were measured in a headless `flutter test` environment, counting CPU time for parsing, building and layout, excluding GPU rasterization. So read them as magnitudes and relative differences, not as real-device frame rates. On a real device you also pay for font shaping and image decoding, so it only gets worse.

The three pitfalls are three facets of one thing: all of them pay repeatedly for content already processed. Full re-parsing re-parses. One widget for a whole reply rebuilds. Notify-per-chunk refreshes. What needs fixing is not three pieces of code but one habit.

One unavoidable piece of context: many people reach for flutter_markdown and hand Markdown to the official package, but it is unmaintained on pub.dev. The last release stopped at 0.7.7+1 in May 2025, and the package page points to flutter_markdown_plus as the successor. On streaming rendering, the official path is currently broken, and the community and every client vendor are writing their own.

How do you handle streaming Markdown? Chunk-by-chunk parsing or full re-layout? Tell me in the comments.
