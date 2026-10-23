---
title: "What Comes After CLAUDE.md: Don't Write One for a New Project"
date: 2026-10-12T08:03:05+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "A bit over a month ago, Claude Code refused to support AGENTS.md — the industry convention already adopted by plenty of projects and coding agents — and Shopify CEO Tobi Lütke publicly warned he was ready to drop it. Anthropic's reasoning at the time sounded reasonable: different models need different context and instructions."
cover:
    image: "/images/claude-md/cover.png"
    alt: "What Comes After CLAUDE.md: Don't Write One for a New Project"
    relative: false
---

## First, something odd

A bit over a month ago, Claude Code refused to support AGENTS.md — the industry convention already adopted by plenty of projects and coding agents — and Shopify CEO Tobi Lütke publicly warned he was ready to drop it. Anthropic's reasoning at the time sounded reasonable too: different models need different context and instructions.

A month and change later, the Claude Code team confirmed it will support AGENTS.md. But Thariq Shihipar, who owns that area on the team, went further on the Latent Space podcast:

> I think, in the limit, CLAUDE.md disappears. And maybe we don't even have to wait that long — I think if you start a new project now, you're probably better off with no CLAUDE.md at all.

## His four reasons

The first: the model's floor is rising:

> As models get better, the floor on the simplest tasks keeps going up.

The second is the most practical one, and it's about order. Let it run first. Only when you keep seeing the same failure mode do you add the rule.

The third is counterintuitive:

> If you keep this ever-growing log of failure modes around, chances are it over-constrains Claude.

The fourth is maintenance cost. This varies by model — you might need FABLE.md, you might need OPUS.md, and it can even differ between two versions of the same model. His words: maintaining different files is just painful.

## So what should you write

In the interview the host, swyx, pointed at two things and said they matter too much to live only in a prompt. One is Goals, where the project is headed. The other is external artifacts that outlive a single session, things like decision logs and experiment logs. Thariq acknowledged in his answer that they still have to do AGENTS.md, but in the same breath he called maintaining multiple files the most painful part.

Follow that logic and the convention file's role shifts from prohibition list to goals and record. Tell the model where the road leads. Keep the judgments you've already made in the file, instead of stacking twenty don'ts at the top. A prohibition list is a patch over the boundaries of an older model. Goals are not.

## Local test: how convention files actually get loaded

I switch between several agent clients every day, so I went through the loading rules on this machine. Four things are worth writing down.

First, AGENTS.md support is recent. I went through the official changelog and it shows up in 2.1.277:

> Added AGENTS.md support: in a project with no CLAUDE.md, Claude Code reads AGENTS.md instead; change it under "Project instructions" in /config

Note the positioning: in a project with no CLAUDE.md, Claude Code reads AGENTS.md instead. It's a fallback, not a peer. And the build installed here is 2.1.150, older than that, so Claude Code on this machine still doesn't recognize AGENTS.md.

Second, another client — I use Hermes daily, the thing talking to me in Feishu right now — takes a completely different route. It scans candidate files in the current directory by priority, lets one win per tier, and marks the rest shadowed:

> not loaded — higher-priority context type wins

Third, it flatly refuses to read files inside other tools' directories. The source says it plainly: subdirectory hints reject paths outside the working directory, so ~/.codex/AGENTS.md and ~/.claude/CLAUDE.md never leak in. The ~/.claude/CLAUDE.md on this machine is only 226 characters (one /graphify routing rule), and Hermes has never read it.

Fourth, loading has a ceiling. Subdirectory hints carry at most 32,000 characters, truncated head and tail beyond that. My project's root file is 36,631 characters, so the back half has been getting cut off.

## Do the math

I ran the local agent's own tokenizer estimate over it:

```
   7151 chars   ~  1794 tokens  ~/AGENTS.md
    226 chars   ~    57 tokens  ~/.claude/CLAUDE.md
   9387 chars   ~  2357 tokens  ~/kmp-shop/CLAUDE.md
  36631 chars   ~  9234 tokens  ~/.hermes/hermes-agent/AGENTS.md

AGENTS.md in the hermes-agent repo: 12 files
169,982 chars total (about 42,496 tokens)
```

One repo, 12 convention files, over forty thousand tokens between them. It's actually written with restraint — the root file itself says only global universal stuff goes here, each directory has its own AGENTS.md, target 8k characters — but the number is what it is, and you carry all of it every time you start work.

![](/images/claude-md/fig-read.png)

## My call

Thariq is saying don't hoard, not don't write. Those two keep getting retold as one thing, and then everyone either deletes every file or keeps piling more in.

My own test is one line. If this rule corresponds to a specific failure, keep it. If it only corresponds to "I'm worried it'll do the wrong thing", that's a guess, don't write it yet. The first is a mistake the model actually made. The second is me imagining boundaries on the model's behalf.

There's also the layering. Don't stuff repo-wide rules into one root file — that's exactly where my 36,631 characters came from. What actually works is a small file per directory, and whoever works in a directory loads that one.

A ready-made example: this root file says target 8k characters, and it grew to 36k. Convention files need someone deleting things on a schedule, or they get fat on their own.

When was the last time you deleted something from your project's convention file? Comments welcome.
