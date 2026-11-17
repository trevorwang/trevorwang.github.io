---
title: "DeepSeek Input Is 50x Cheaper, but You May Never Hit the Cache"
date: 2026-11-17T12:29:50+08:00
draft: false
categories: ["WeChat"]
tags: ["DeepSeek", "AI Agent", "Caching"]
summary: "A cache hit costs 0.02 yuan and a miss costs 1 yuan per million tokens, a 50x gap. I measured it locally: the hit rate is roughly the share of tokens before your insertion point, so inserting at the start keeps 7% while appending at the end keeps 97%."
cover:
    image: "/images/deepseek-cache/cover.png"
    alt: "DeepSeek Input Is 50x Cheaper, but You May Never Hit the Cache"
    relative: false
---

DeepSeek's current price list splits deepseek-flash input into two columns: 0.02 yuan per million tokens for a cache hit, and 1 yuan for a miss. Fifty times over.

Everyone is sharing that number, but few stop to ask one question: am I actually hitting it.

I measured the whole thing from scratch. The answer is harder to pin down than I expected, because whether you hit or miss is not entirely up to you. It depends more on whether your sentence goes at the beginning or at the end, and the docs mention that in a single throwaway line.

## First, how the price list actually reads

The official pricing page now puts both models side by side. deepseek-flash and deepseek-v4-pro list input in two columns, cache hit and miss. The original text looks like this:

| | Cache hit | Miss |
| :--- | ---: | ---: |
| deepseek-flash off-peak | 0.02 yuan | 1 yuan |
| deepseek-flash peak | 0.04 yuan | 2 yuan |

Output is 4 yuan per million (off-peak). The 13.5 yuan tier belongs to deepseek-v4-pro. They are different models, and the input to output ratio is 1 to 4 for flash and 1 to 3 for pro.

Note that off-peak is exactly half of peak. Peak hours are 9 to 12 and 14 to 18 on weekdays. Everything else, all day on weekends and public holidays, is off-peak. So running batch jobs at night or on weekends cuts the price in half. That is easier to fully exploit than caching, and it requires no code changes.

![DeepSeek current price list with the cache price gap](/images/deepseek-cache/fig1.png)

## A single request already shows the hit breakdown

In the response from `/chat/completions`, the usage field is more detailed than many people assume:

```json
{"prompt_tokens": 15520, "completion_tokens": 30,
 "prompt_cache_hit_tokens": 15360, "prompt_cache_miss_tokens": 160}
```

Three requests with the same long prefix, only the trailing question changed. The first was a cold start, hit was 0, and all 15,520 tokens were billed at 1 yuan. From the second request on, hit became 15,360 and only 160 tokens counted as a miss.

That is a 99% hit rate, and input cost drops from 1 yuan per million to around 0.02. Those 160 missed tokens are the trailing question itself, unavoidable overhead on every call.

Many SDKs do not pass this field through, so you have to read it from the raw response. If you use a wrapped client, first confirm it preserves `prompt_cache_hit_tokens`. Without it you cannot even tell whether you are hitting the cache, and every optimization discussion after that is pointless.

## But the hit check is stricter than you would think

This is the most surprising part of the whole exercise. I ran a controlled experiment with a prefix of just over seven thousand tokens. I built a cache once, then inserted one new sentence at different positions to see how much of the hit survived.

| Insertion point | Hit tokens | Miss | Hit rate |
| :--- | ---: | ---: | ---: |
| At 10% | 512 | 7273 | 7% |
| At 25% | 1664 | 6121 | 21% |
| At 50% | 3712 | 4073 | 48% |
| At 75% | 5632 | 2153 | 72% |
| At 90% | 6784 | 1001 | 87% |
| Appended at end | 7552 | 233 | 97% |

The hit rate is almost exactly the share of tokens before the insertion point.

That means the cache matches segment by segment along the prefix. Whatever you insert in the middle invalidates everything after it. Insert at the 10% mark and you keep only 7%. Insert at the end and you keep 97%.

Many people assume that changing a little content only affects that little bit. In reality everything after that line has to be recomputed.

![The effect of insertion point on hit rate](/images/deepseek-cache/fig2.png)

Conversely, appending the same sentence to the end of the conversation keeps the hit rate above ninety percent, and the cost is almost the same as the previous call. Same edit, two positions, more than a tenfold price difference. The docs do not say this outright, so you have to measure it yourself.

## This rule decides how you organize the prompt

Since prefix matching is segment by segment, the things that change should go later and the things that stay fixed should go earlier. That runs exactly counter to the instinct for writing prompts.

The counterexample is the most common pattern: the user's current question goes into system, and project conventions and codebase context follow behind it. Now every new question changes the very first sentence of the prefix, and the hit rate goes to zero.

![Comparison of two concatenation orders](/images/deepseek-cache/fig4.png)

The good pattern is the opposite: fixed content first, question at the end.

I ran a more realistic comparison. The same 15,520 token prefix, 100 consecutive runs, only the trailing question changed each time, the prefix untouched. This is the most common everyday usage: in a single conversation only the last line is new and everything before it is existing history.

With caching, 100 runs cost 0.0738 yuan in total. Without caching, 1.5640 yuan, a saving of 95.3%. That number is far from the fiftyfold I expected, and the reason is the next sentence.

Note that 95.3% is not fifty times, because output never participates in caching. The 30 output tokens cost 4 yuan per million every time. The longer the prefix and the shorter the output, the closer the saving gets to the theoretical fifty times.

![Cost comparison across 100 requests](/images/deepseek-cache/fig3.png)

## How long does the cache live

I repeated the same prefix after 28 seconds, 90 seconds, and 150 seconds. The hit rate stayed steady at 96% with no decay.

The official docs describe retention in hours. The exact duration depends on two things: whether the prefix was used recently, and how much cache pressure exists at that moment. Frequently used prefixes are basically fine. Ones you run once and leave idle need attention.

My test method was conservative and only verified the short term. If you really run long term, make sure the same prefix gets reused repeatedly within a day. Sitting idle is itself what makes caching meaningless.

One easily overlooked point while I am here. Caching only kicks in above a minimum effective prefix length. Very short prefixes are not worth caching at all, and the money you save on those tokens will not cover the effort of writing a few extra lines of code. You only start to feel it at a few thousand tokens, and past ten thousand, hitting or missing means two different price brackets.

## Three conclusions you can copy directly

The hit rate is computed segment by segment along the prefix. Where you insert a sentence decides how much you keep. If you must change something, change the end.

Peak pricing is double off-peak, and weekday daytime counts as peak. Batch jobs you can move to the evening or the weekend save half, with no technical change, no line of code touched, just a different time slot.

Output never participates in caching, and that is the easiest item to get wrong when estimating savings. If your workload pairs a large prefix with long output, the fiftyfold cache discount gets diluted by output cost, and you need to work out the real saving ratio yourself.

## My take

DeepSeek's price page states cache hits clearly, but nobody tells you how to earn one.

The real barrier is not the way you call the API. It is how you organize that block of text. A stable prefix is designed, not tuned. It lives in your code structure, not in parameter tweaks.

With a casually concatenated prompt, the cache hit column will always be zero.

So the first thing to do is not optimize. It is to look at `prompt_cache_hit_tokens` in the response. If that number is 0, you are paying full price right now, have not captured a cent of the fiftyfold discount, and may still believe you are already saving money.
