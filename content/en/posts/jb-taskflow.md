---
title: "Split One Developer Request Into Three Messages and the Agent's Cost More Than Doubles"
date: 2026-10-30T11:46:11+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent", "JetBrains", "Benchmarks"]
summary: "JetBrains analyzed 4,782 real IDE sessions: splitting one request into three messages more than doubles cost and barely moves the resolve rate."
cover:
    image: "/images/jb-taskflow/cover.png"
    alt: "Split One Developer Request Into Three Messages and the Agent's Cost More Than Doubles"
    relative: false
---

JetBrains researchers went through 4,782 real sessions, all of them engineers talking to a coding agent inside their own IDE, about 14,000 user messages in total.

The conclusion in one line: the benchmarks you use to grade agents today aren't measuring how people actually use them.

In SWE-Bench Pro, 93.8% of tasks are feature additions and bug fixes, and every task arrives as one complete message. The agent sees the whole picture at once and then starts working.

In real sessions, feature additions are only 19%, bug fixes 26%. Together they don't add up to half.

So what's the rest? 6% asking how the code works, 10% general programming questions, plus planning, code review, refactoring, running commands, debugging. None of that exists in SWE-Bench Pro, because that benchmark's task definition has no such action as "asking a quick question."

![The same issue, two different user profiles](/images/jb-taskflow/fig1.png)

## Half the sessions are a single message

They counted every session by number of user messages.

50.9% of sessions have just one message, 16.2% have two, 8.8% have three, and everything from four up to five adds up to under 15%.

![4,782 real sessions by message count](/images/jb-taskflow/fig2.png)

So the rest of the analysis only looks at the 33% with at least three messages, which they call long sessions — a single message is a one-shot Q&A. Inside long sessions, the breakdown looks like this: 6% asking about the project's own code, 10% general programming questions, and the rest scattered across planning, review, refactoring, debugging, execution — feature additions and bug fixes have been diluted down to a small slice.

Run SWE-Bench Pro through the same classifier and you get 55% feature additions, 39% bug fixes, 6% refactoring. That's it.

## Three public corpora, three completely different patterns

They also ran the same labeling on three public agent conversation sets.

SpecStory is a public export of IDE sessions. 521 of its 1,000 are long sessions, the closest to their own data, with a noticeably high share of planning.

DataClaw is a community-shared set of sessions, 585 long ones, almost all execution and short commands. SWE-chat is records of real users in a chat interface, 698 long sessions, predominantly execution and planning.

The three datasets come from different products, different users, different collection methods, so they don't compare volumes. But one thing is hard fact: calibrate a benchmark against any one of them and you get a different task mix, and that difference isn't something sample size can wash out.

Not even their own 4,782 count as ground truth. In their words, you have to pick a target first and then declare it; there is no default one.

That sentence is the whole skeleton of the study. Everyone used to assume "building tasks from GitHub issues is what's real." This paper says no — issues are just one slice of one population. Swap the population and the data takes on a different shape, and nobody is in a position to call their own slice the standard.

## They didn't build a new benchmark, they retrofitted the old one

Building an interactive benchmark from scratch means throwing away all the repos, environments, and tests SWE-Bench has accumulated over the years. So they took another route: keep the tasks, repos, tests, and original requirements, and only rewrite the user messages.

The method is to split each issue into three ordered requests, with zero invented content allowed. Each request has to be actionable against the code state the previous one left behind, identifiers and code snippets stay verbatim, and after splitting another model checks for any dropped requirements. Of the 731 public SWE-Bench Pro issues, 31 failed validation, leaving 700.

Those 700 are the entire battlefield. Every number from here on comes from them.

The original issue is roughly a thousand words in one message, all crammed into a single block. After splitting it looks like this — the first sentence gives only the main symptom, edge cases wait for the second and third.

qutebrowser had an issue where LinkedIn wouldn't load, caused by an older QtWebEngine missing `Array.prototype.at`.

The first request just says "fix LinkedIn failing to load on macOS with QtWebEngine 5.15.2, and catch it with a polyfill."

Only the second one adds "it must cover versions before 6.3 that lack native support, and both positive and negative indices have to be correct." The third only then says "out-of-range must return undefined, and only inject when the native method is missing."

Line the three up and the complete spec from one message has been spread into three rounds of progressive disclosure.

## They also injected "ask about the project itself" questions

In real sessions, 6% is asking how the project's code works, and issue benchmarks have none of that. So they had a model read the repo plus a trajectory where the problem had already been solved, and generate one to three questions.

Each question carries two things: a reference answer, and an evidence script that actually runs. During dataset construction it's run against a clean repo — if it produces no evidence, or it modifies the repo, the question is dropped.

That rule conveniently eliminates a whole class of questions: ones whose answer isn't in the code ("why was it designed this way back then") are physically impossible to produce.

The qutebrowser task later got a companion question whose answer named two config entries and touched five files.

## After splitting: cost doubles, scores don't move

Here's the most valuable part of the whole paper: four models, 700 tasks, three feeding modes, three runs of each.

Single-shot means throwing the original issue over as one message. Split means sending the three requests in sequence, with nothing added in between. Split plus Q&A means also adding those repo questions, with an optimizer choosing where they go.

The result: splitting raised each model's cost 2.0 to 2.6 times.

Same repos, same tests, not a single added requirement — all the extra money went into "requirements arriving in batches."

Scores barely moved. GPT-5.6 Terra stayed flat; the other three dropped 4 to 5 points on average, roughly the same as the variance from running the same setting three times.

![Measured results across four models × three settings](/images/jb-taskflow/fig3.png)

They also ran a control: splice the three rewritten requests back into one message and send that. All three GPT models' cost and scores landed right back in the single-shot bucket. So the expense is in the act of sending three times, not in the rewording.

## Splitting gives you intermediate checkpoints

Run one long requirement to the end and you get a single final score, with right and wrong smeared together — if you want to know which step collapsed, you have to go dig through the trajectory yourself. Split into three steps and you can run the tests after each one, scoring only against the requirements revealed so far, and the failing step is visible at a glance.

In an ablation over 64 tasks, the pass rate and test pass rate climbed with each request. One problem becomes several verifiable checkpoints, and diagnosis gets much clearer.

This is probably the only benefit of splitting. Beyond it, splitting is pure cost.

## What it hasn't solved

They only generated two kinds of messages: split requirements, and questions about the repo. Planning, review, execution, and debugging without a fix, all of which show up in real sessions, still have no generator — and for most of them, even "how do you automatically judge correct from incorrect" hasn't been worked out.

One more thing: every message is frozen before the run starts. Reproducibility is real, and so is the fact that the "user" never changes their mind in response to the agent.

The research stops there. The things they listed as not done go on for a while, right down to admitting a weakness like "the classifier has no human-labeled accuracy set."

## What this means if you write code

When you look at a model leaderboard, you assume that number means "how good it is at writing code." What this paper says is that the number also contains something you've never been choosing: the pace at which it's asked to work.

Same model, requirements delivered all at once versus in three batches — more than double the cost, roughly the same score. So how you ask for things on your own project is a separate question, and it's the one you control. My habit is to write background, goal, and constraints all in one message. Reading this hasn't changed my mind. When cost actually becomes the sticking point, the first thing I look at is how many extra steps it took, not whether it got the answer right.

One fun corollary: if every benchmark measures "everything delivered at once," the models being trained will get used to that pace too, and then nobody can measure the follow-up questions, the course corrections, the progressive disclosure that real sessions are full of.

Think back to your last few agent sessions — how many were solved in one message, and how many took several rounds back and forth?
