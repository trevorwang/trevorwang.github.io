---
title: "A 45,000-Star Security Audit Skill Spends Half Its Code Stopping AI From Making Things Up"
date: 2026-10-25T12:36:03+08:00
draft: false
categories: ["WeChat"]
tags: ["Security", "AI Agent", "Code Review"]
summary: "1,778 lines of documentation, 3,498 lines of validation. The heaviest part of Cloudflare's open-source audit skill is not finding bugs, it is stopping the agent from inventing them."
cover:
    image: "/images/cf-security-skill/cover.png"
    alt: "A 45,000-Star Security Audit Skill Spends Half Its Code Stopping AI From Making Things Up"
    relative: false
---

There is a repository on GitHub, cloudflare/security-audit-skill, with over forty-four thousand stars. Its job is to turn a coding agent into a security auditor. Read only the README and you would think it is another prompt collection, a pile of attack techniques for the model to check off.

So I pulled the whole repository down and counted, and that is not what it is.

The documentation is 1,778 lines, fourteen attack-surface manuals plus the process writeup. The validators themselves are 1,645 lines, with 142 explicit error checkpoints across the two of them. Add 1,392 lines of tests and 461 lines of schema and the validation side totals 3,498 lines.

Documentation for finding bugs: 1,700. Code for stopping it from making things up: 3,500.

![The full repository across 5,276 lines, the validation side roughly twice the documentation](/images/cf-security-skill/cas-lines.png)

## The funnel they publish

The Cloudflare blog post about their own vulnerability harness has a set of numbers.

The discovery stage generated 20,799 raw candidates. 12,057 survived independent validation. Merged with findings from another harness, the central pool reached 13,841. Deduplication collapsed 5,442. Another 1,154 were routed out as wrong-repo or low-risk and fed back into the system.

What reached engineers: 7,245. From 20,800 down to 7,200, six and a half out of ten dropped.

![The published funnel: 20,799 raw candidates flowing down to 7,245](/images/cf-security-skill/cas-funnel.png)

They write it plainly in the post. The core metric of this harness is not recall. It is keeping the number of unconfirmed findings in front of real humans as close to zero as possible.

Their words: the architecture has to be a relentless filtering funnel.

## Two models checking each other

One line in that post I read several times.

They run discovery on one model and validation on an entirely different one. The reasoning is spelled out. If you make model B judge model A's output, B looks at it through a completely different set of weights and training data, which makes it an unbiased adversarial party whose only job is to tear apart A's assumptions.

The same post has another line worth quoting. Point different models at the same target and each turns up a different share of the bugs.

There is also a number buried in the body text. In their own testing, a single run found roughly half the vulnerabilities that repeated runs found in total.

Half. They said that themselves.

![Discovery on one model, validation on another, each tearing apart the other's assumptions](/images/cf-security-skill/cas-twomodels.png)

## I went at the validators with bad data

Documentation alone proves nothing, so I made the code run.

Its bundled tests total 65 across the two validators, all passing, zero failures. The test cases are interesting in themselves: rejecting Unicode format controls, rejecting a FIFO without blocking, rejecting a symlink without following it, rejecting deep nesting without blowing the stack, working under a constrained heap.

Then I wrote data designed to hit it.

I wrote a needs_validation record the way AI most typically cuts corners, and threw in a severity field for good measure. The validator came back with two errors. One said the field should not exist, the other said a record of this kind must not contain severity.

The same mistake blocked from two directions at once.

I wrote the trace field as a single sentence of prose with no concrete location. The validator told me it expected an object and got a string.

The most humbling part was the first attempt. I thought I had read the schema, and my very first submission was wrong in four places. The trace kind can only be entrypoint, propagation or sink, and I wrote source.

An agent that patiently read all 461 lines of schema still got it wrong four times on the first try. This is not nitpicking. It is simply not fakeable.

![Structured errors for four typical mistakes: severity out of bounds, prose trace, missing fields, wrong enum value](/images/cf-security-skill/cas-validator.png)

## What it handed back was an empty array

The validators are the outer layer. The real question is what an agent produces when you point this standard at a file.

I picked a target I could check against, the 773-line validator itself. The instruction was a single line: every candidate must supply six elements, drop any that is missing even one, do not pad the count.

It ran for two and a half minutes and handed back an empty array.

Zero candidates.

But it did not stop there. It listed ten things that looked like vulnerabilities and explained, one by one, why they are not. A few you can check yourself.

Terminal injection. Validation failures print the user-supplied path to stderr, which looks like ANSI escape injection. It traced line 98 and found every message passes through an escape function covering control characters, format characters and surrogates. It also noticed safeQuote clips to 256 characters before escaping, and that clipping can split a surrogate pair in half, and that half a pair is itself inside the escape range.

Resource exhaustion. The 5 MiB limit is checked twice. Once with fstat on the file size, once with a running byte counter during the read, so a file that grows mid-read still gets caught. Depth 64 and array size 1000 are likewise checked both on the raw text and on the parsed value.

Symlinks and FIFOs. The file is opened with O_NOFOLLOW and O_NONBLOCK, and if the platform lacks either constant it refuses to run rather than degrade. It then fstats the descriptor rather than the path, so there is no open-then-reopen race.

Two more in a sentence each. The ReDoS regex compiles a pattern from the bundled schema, not from findings.json, so the attacker controls the subject string and not the pattern. Every attacker-controlled key is read through a hasOwn guard.

None of the ten held up. Then it closed with four engineering observations, clearly labelled as suggestions rather than vulnerabilities: exit code 1 is overloaded so unsafe input and failed validation are indistinguishable to a caller; the catch discards the exception; the schema regex is recompiled per string; UNSAFE_DIAGNOSTIC_CHARACTER carries the /g flag.

![The agent returned an empty array, plus ten point-by-point rejections](/images/cf-security-skill/cas-empty.png)

## Why an empty result is more convincing than a finding

An agent required to supply evidence will hand in a blank page when the code is watertight, rather than invent a finding.

I will not overstate that. I have one sample, and this code was written by a security team. It does not prove the workflow finds real bugs in genuinely vulnerable code.

What it does prove is the other half. The workflow stops things that look like problems from being packaged as vulnerabilities.

And that half is the genuinely hard part. Finding things is not hard, models love to talk. The hard part is getting them to shut up when they should.

## The three walls they hit

The blog post also covers what broke when they went from a local skill to fleet-wide scanning.

The first wall is context exhaustion. An hour in, the window fills and the model starts eating its own memory, and the bug it chased all morning is gone. Their fix is a hard engineering constraint: each agent's job is cut extremely fine, keeping context below 25 percent of the total window. Fill it and the model starts hallucinating, which is the original wording.

The second wall is speed, which means running in parallel. They found some models do not proactively fork sibling agents, so they added a request mechanism: hit a path outside scope but worth pursuing, and you can fork a sibling. That accounts for 9 to 20 percent of all tasks.

The third wall is the most mundane. One rate-limit error or a connection hiccup and hours of work are gone.

## Same battlefield as verification

Last month the Flutter team published a post whose core claim was that code generation stopped being the bottleneck, and verification is the bottleneck now.

Writing about Compose last week, what I saw was JetBrains opening the running application to an agent so it can change code, reload, screenshot and confirm on its own.

Cloudflare takes a third path. It does not optimize the speed of verification. It turns verification itself into a gate you cannot slip past.

What all three have in common is clear. What is scarce is not the ability to make AI do work, it is the ability to judge whether the work is right. And on that second half, the mainstream approach is still handing judgment back to another AI.

## How to use it

The skill installs directly. The README says npx skills add, and it works in any agent skill directory as long as your agent supports parallel sub-tasks.

One caveat. It requires an OS-enforced sandbox: no external network, empty environment, read-only target, writes only to its own scratch directory, explicit resource limits. Without that, the skill refuses to execute target code and downgrades every conclusion to needs-validation.

I originally meant to run all six phases in my own dsh and got stuck on credentials, since headless mode cannot reach the API key. I ended up running the discovery phase with an independent agent, and the validation phase never started because there were zero candidates. So the numbers here come from three sources: the funnel from the Cloudflare post, the tests and validator experiments from my own machine, and the audit result from one independent agent.

In your team's code review today, is a human reading AI-written code, or is AI reading yours?
