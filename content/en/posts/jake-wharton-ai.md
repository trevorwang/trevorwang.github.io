---
title: "A Kotlin Legend Turned Down 80% of the Job Market: Why I Flatly Refuse to Use AI for Code"
date: 2026-09-17T07:24:40+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "If you are a senior engineer, what do you write down as your requirements when you go job hunting?"
cover:
    image: "/images/jake-wharton-ai/cover.png"
    alt: "A Kotlin Legend Turned Down 80% of the Job Market: Why I Flatly Refuse to Use AI for Code"
    relative: false
---

## A job-hunting criterion that makes no sense at first

If you are a senior engineer, what do you write down as your requirements when you go job hunting?

More money, less work, a modern stack, a solid team. Roughly that.

Jake Wharton is different. This Android / Kotlin open source legend, known to Chinese developers as "J God," drew up nine criteria for choosing a job.

**The first one: do not join an AI company, do not work on AI-centric products, and do not accept a company forcing AI on its engineers.**

By his own estimate, that single line rules out about 80% of potential employers.

Before you rush to call him precious about it: it is not that he has no options. A senior Kotlin engineer in the US typically starts at a $200K base salary, plus roughly $100K a year in equity vesting over four years. Going into AI could pay more.

He did it anyway.

InfoQ published the full transcript of this two-hour interview a few days ago, over thirteen thousand characters. What struck me most was not his conclusion. It was the reasoning he gave.

## Who he is: one of the people who made Kotlin Android's official language

If you are not in the Kotlin world, you may not know how much weight that name carries.

Jake Wharton is the author or core contributor of OkHttp, Okio, Retrofit and other star open source libraries. Anyone who has written Android has probably used his code.

More importantly: **he was the first engineer on the Google Kotlin team.**

Before Google, he worked at Square (later Cash App). To convince the company to adopt Kotlin internally, he wrote an argument document. He later made it public, and it indirectly drove the whole Android community's interest in Kotlin.

Then Google decided to make Kotlin an official first-class language for Android, and he joined that team.

He says the opportunity was "almost impossible to turn down." Nothing is a bigger endorsement than joining the company responsible for the entire Android toolchain and pushing Kotlin to official status.

**So he is not anti-technology. He is someone who bet his career on a language and a toolchain.**

Which raises the question: why is a person like that so resistant to AI coding?

## Reason one: AI spreads in the opposite direction from every tool before it

This is the passage I agreed with most. Wharton's words:

> Traditionally, a development tool is usually pushed bottom-up by engineers. Maybe one engineer falls in love with a new language, Kotlin for instance. Or a new IDE, IntelliJ IDEA. Or a new build system. The engineer finds value in it at work, the enthusiasm spreads to others, and then everyone goes and convinces engineering management to approve adopting it.
>
> But AI tools seem to spread in the exact opposite direction. Why is a company CEO telling engineers what tools they should use in their daily work?

That is a sharp observation.

Think about how Kotlin took off: **a group of engineers genuinely found it good, started using it quietly in their projects, and slowly convinced their bosses.** Bottom-up. Organic.

The rollout path for AI coding tools at plenty of companies looks like this: **executives see the news, mandate that everyone must use AI to boost productivity, and put it in performance reviews.** Top-down. Administrative.

**The legitimacy of a tool used to come from it actually being good. Now, sometimes, it comes from being mandated.**

That shift is worth a thought from every engineer.

## Reason two: what you hand over is capability, not just work

On the idea that AI turns engineers into cogs, here is his full formulation:

> AI can help engineers handle repetitive work, but the most irresponsible use is letting AI generate code beyond the user's own capability. If you cannot complete the task without a large model, you often cannot truly understand, verify or maintain the result either.
>
> Once dependency forms, what an engineer hands over is not just part of the work but their own capability and bargaining power, and eventually they may become a cog that is easier to replace at low cost.

Note the boundary of that statement: **he did not say "do not use AI." He said "do not use AI to generate code beyond your capability."**

Those are very different sentences.

The first means "AI will replace you." The second describes a more specific and much quieter risk:

If a requirement would normally take you two days to think through the design, and now AI spits out working code in ten seconds that you do not fully understand —

**you have the code, but your understanding of the system has actually degraded.**

A few rounds of that, and part of what you maintain is something you cannot read yourself.

At that point, where is your value anchored?

## Reason three: copyright and licenses, the accounting is not settled

He is direct about this one:

> As I see it, they are built on large-scale use of copyrighted content. Model training ignored artists' copyright and ignored open source licenses.
>
> Being able to access something on the internet does not mean you have the right to recombine it, rewrite it, or output it verbatim.

For someone who has poured a lot of himself into open source, the position is easy to understand. **His code was taken to train models, and then what the models produce is sold back to him. Ethically, that accounting really is not settled.**

He adds a detail: open source maintainers now have one more burden to deal with — **a flood of low-quality AI-generated PRs.**

I believe that. Anyone who has maintained an open source project has felt it.

## But he does not think AI is going away

Do not misread him. His read on AI's capabilities in the interview is actually restrained and clear-eyed:

> "AI is already unnervingly powerful," but that does not mean it can carry the whole software engineering process reliably and without error.
>
> To him, AI can at most be one tool in the toolbox. It cannot become the foundation of engineering capability.

"Not the foundation." I think that metaphor is exactly right.

**A tool in the toolbox is something you can leave unused. A foundation is something you cannot stand without.**

He also makes a counterintuitive point: AI being able to write code does not mean open source libraries lose value. Because the value of open source is not just "here is a piece of code that works." It is **the long-term dividend of continuous maintenance** —

> Someone may fix an edge-case bug today that you would not hit yet. Two years later, when you refactor your business, you would have triggered it, but because you use the same open source library, it was already solved ahead of time.

The logic is tight: **AI gives you one copy of code today. Open source gives you continuously evolving public infrastructure.** Those are not the same thing.

## Kotlin talk, the parts that matter to us

The second half of the interview is about Kotlin, and a few points are genuinely useful if you write Kotlin:

**1. "Kotlin looks a lot like what Java wants to be in ten years"**

His explanation: Java has to avoid breaking existing code, so the language evolves slowly. Kotlin can hand developers many modern features that might land in Java later, and still works with codebases stuck on Java 8 / 11.

**2. Knowing Kotlin is no longer a career advantage**

That stings, but his explanation is fair:

> I am not sure it counts as an advantage anymore. In a sense that reflects Kotlin's success: it has become a default requirement rather than a scarce, hard-to-find skill.

Because Google made Kotlin the official Android language, people learning Android learn Kotlin directly, and employers assume you know it.

**But —**

> If a developer has more than routine Kotlin, things like Kotlin Multiplatform that are relatively advanced, that can still be a differentiator and set you apart from other Kotlin developers.

**"KMP is still a differentiator"** is probably the most direct signal in this interview for people like us.

**3. Meta donated its Kotlin formatter**

In the same week he recorded this episode, Meta contributed its Kotlin formatter to the Kotlin Foundation. It may become the official formatter.

**4. In 2026, Kotlin is governed by three parties**

The ideal is JetBrains, Google and the community all participating. One of the reasons the Kotlin Foundation exists is so the language is no longer wholly owned by JetBrains — so no single company controls it.

## My own bind: building an AI client while listening to him argue against AI

A personal note.

I am writing a booklet about **AI chat clients** right now — building a cross-platform AI chat app from scratch. The stack is KMP.

So reading this interview gave me a strange sense of dislocation: **I am building a product on AI capabilities while one of the most respected people in the industry says AI is ruining engineering.**

Calm down and think about it, though, and I do not find it contradictory.

**What he opposes was never "the tool called AI." It is three things:**

1. Treating AI as mandatory organizational discipline rather than an optional tool
2. Using AI as a substitute for thinking, losing your ability to verify and maintain
3. Business models built on copyright and license problems

Of those three, only the second is something I have to stay vigilant about — especially when I write code and write tutorials myself.

**Tools will keep getting stronger. That is the trend, you cannot stop it and you do not need to. But "is my judgment still in my own hands" is something only you can be responsible for.**

That is probably what he means by "not the foundation."

## Last

One more line from the interview stuck with me:

> I do not think the overall quality of software has improved over the past few years, and I do not think these tools will improve software quality in the long run. But some company executives see a different picture.

Which picture is right? Nobody knows yet. It may take five years to find out.

**But one risk is certain: if a person outsources all of their capability, then whether or not AI is that strong, they have already lost their bargaining power.**

What do you make of it? How much of your current work involves AI?

Tell me in the comments — I especially want to know if anyone else is building products with AI while quietly uneasy about where its boundaries are.

---

*Sources: [full Jake Wharton interview (InfoQ, 2026-09-11)](https://www.infoq.cn/article/QPRcUdPGic5r8Ws8R1gM) · [original video](https://www.youtube.com/watch?v=9WjPdgOK-68)*
