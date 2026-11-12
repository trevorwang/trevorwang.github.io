---
title: "DeepSeek Harness Code Review: The Bottleneck Was Never Git"
date: 2026-11-12T09:20:00+08:00
draft: false
categories: ["WeChat"]
tags: ["DeepSeek Harness", "Git", "AI"]
summary: "DeepSeek Harness 0.2.2-rc.2 added review by branch, commit and file type. I measured every stage on a 21,711-file repository: creating a worktree takes 1.5 seconds, producing a 28,458-line diff takes 75 milliseconds, and what actually does not fit is 300,000 tokens of context."
cover:
    image: "/images/dsh-worktree-review/cover.png"
    alt: "DeepSeek Harness Code Review: The Bottleneck Was Never Git"
    relative: false
---

DeepSeek Harness shipped 0.2.2-rc.2, and one line in the notes said Git Worktree now supports reviewing changes by branch, commit and file type.

I went to upgrade that same night. npm did not have that version at all.

## Three places, three version numbers

GitHub Releases had dsh-v0.2.2-rc.2, published at 11:33 on the 11th. I tried to install it and npm returned notarget.

A quick look at dist-tags: both latest and next still pointed at 0.2.2-rc.1.

![Three places, three version numbers](/images/dsh-worktree-review/fig1.png)

The desktop app was older still, sitting on 0.2.1-rc.2. So my machine was holding three version numbers at once and none of them was rc.2.

That is not a DeepSeek Harness flaw, every release process behaves this way. But it decides when you can actually use a new feature: either wait for npm to catch up, or switch the desktop app to the dev update channel, which is itself something rc.2 introduced.

## So I reproduced it myself

Since I could not install it, I reproduced what DeepSeek Harness does on my own machine.

Worktrees, comparing by branch, slicing by file type: all of that is git commands underneath and has nothing to do with the client. I used my own repository, 21,711 files and 60,037 commits, as the target, with a 50-file repository alongside as a control.

Starting with the slowest step: creating a worktree.

![Worktree creation time](/images/dsh-worktree-review/fig2.png)

Small repository: 32 milliseconds, three runs at 31, 32, 32.

Large repository: 1.5 seconds, three runs at 1.47, 1.49, 1.51. Rock steady. A 46x gap.

The checked-out worktree occupies 275 MB, but `.git` is shared, so the 840 MB repository was not copied again. That design saves a lot of disk.

1.5 seconds is not long, but if an agent creates a worktree on every edit, with network round trips, disk jitter and a pile of invisible waiting mixed in, that 1.5 seconds becomes a bet that could time out at any moment. The release notes say "relaxed the wait time for creating worktrees in large repositories," and this 1.5 seconds is what that refers to.

## Three review dimensions, all in milliseconds

I made two rounds of changes inside the worktree to simulate an agent at work: 40 files in the first round, 240 in the second, totaling 311 files and 28,458 lines against the main branch.

Then I compared along the three dimensions it describes.

![Timing across the three review dimensions](/images/dsh-worktree-review/fig3.png)

Stats by branch: 65 milliseconds. Full body: 75. Grouped by file type: 29. Log by commit: 26.

All four are under a hundred milliseconds. Nothing at the git layer is slow.

Here is the counterintuitive part. A 28,458-line, 1.27 MB diff takes git only 75 milliseconds to compute. I expected it to dominate and it turned out to be the fastest thing in the room.

## Pick the wrong baseline and you review nothing

I stepped on something here worth writing down.

My first comparison used `git diff main...HEAD`. Three dots means comparing against the merge base of two branches. It printed zero files.

The reason is simple: local main on that repository was far behind, so HEAD was itself an ancestor of main. The merge base resolved to HEAD, and comparing a commit against itself is of course empty.

Switching to `origin/main...HEAD` immediately gave 311 files.

Same repository, same branch, only the baseline changed, and it went from 0 to 311 files.

So the thing to be careful about with review-by-branch is not whether diff is fast, it is what the diff is comparing against. If the agent's branch was cut from an old baseline, you will see considerably less than was actually changed, and nothing will warn you.

Three dots compare merge bases, two dots compare endpoints. That distinction rarely matters until review is automated, at which point it becomes the difference between seeing the change and not seeing it at all.

## How to read those 28,458 lines

Since git is not slow, the slowness must be elsewhere. Split the diff by extension and the answer surfaces.

![Change distribution by file type](/images/dsh-worktree-review/fig4.png)

Python: 187 files, 12,468 lines, 43%. JSON: 52 files, 5,690 lines, 20%. TypeScript: 46 files, 2,647 lines. TSX: 12 files, 1,526 lines.

The interesting part is those 5,690 JSON lines. Lock files, contract snapshots, config exports, a fifth of the total line count, and under five percent of it needs to be read line by line.

The value of review-by-file-type is not tidier categorization. It is having a principled reason to skip an entire 5,690-line folder.

## What a worktree saves is not disk

I assumed worktrees would eat disk space. Measured, that worry was unfounded.

Against 21,711 files and an 840 MB `.git`, the worktree occupies only 275 MB, because `.git` is shared and only the working tree gets copied out.

What those 275 MB buy is isolation: one repository can host several worktrees, each on a different branch, without interfering. Agent A edits code on feat-a while agent B edits elsewhere on feat-b, and each side has its own working tree and index.

A sandbox or a full repository clone costs you 840 MB of disk plus tens of seconds of copying. A worktree costs 1.5 seconds, and throws in branch isolation for free.

That is also why review is built on top of worktrees. The branch is there, the commits are there, the files are there, and whichever dimension you slice by, the raw material is already in place.

## The bottleneck is context, not tooling

Put every stage's number side by side and the conclusion is clear.

![The magnitude of each stage](/images/dsh-worktree-review/fig5.png)

Creating a worktree: 1.5 seconds. Producing the diff: 75 milliseconds.

Hand that diff to a model and it is 1,268,059 characters, roughly 300,000 tokens.

What is 300,000 tokens? Mainstream models now carry context windows around 200,000, so this single change does not fit, before counting repository structure, the task description and your own question.

So when rc.2 says "reduced jank when viewing large changes," what it is janking on is not git. It is the layer that renders this pile, feeds it to a model and puts it in front of a human. No amount of optimizing those millisecond figures helps, because the real ceiling is here.

Review by file type, by commit, by branch: three paths that are three ways of cutting. Cut it down far enough and only then can a model finish reading.

## The one I could not test

rc.2 added two more things: named and multi-level SubAgent collaboration, and AgentTeam that can be enabled per session with shared members and a task panel.

I cannot reproduce that locally, because it is not a git-layer concern, it is orchestration. And rc.2 is not installable from npm, so all I have is rc.1, and running that would not show rc.2 behavior.

So I only write what I verified. That one is for people who can actually get their hands on it.

## What you can do now

If your agent already works on branches, two things are worth changing today. First, give the worktree directory a fast disk. Most of that 1.5 seconds is disk I/O, and a spinning disk or a network volume makes it worse.

Second, read the stats before the body. 312 lines versus 28,458 is a 90x difference, and reading the former first can filter out half the files you do not need to open.

As for DeepSeek Harness rc.2, it is a matter of waiting for npm. Once you have it, the first thing worth doing is running worktree creation against your largest repository to see whether your machine takes 1.5 seconds or 15.

That number decides whether you let it branch freely, and whether your pipeline can run several agents editing the same codebase in parallel. A 1.5-versus-15-second difference on one machine ends up being the difference between "three agents at once" and "one at a time," two entirely different architectural choices.
