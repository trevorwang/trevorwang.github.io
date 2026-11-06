---
title: "JetBrains' New IDE, and the Editor Is Read-Only"
date: 2026-11-04T21:10:00+08:00
draft: false
categories: ["WeChat"]
tags: ["JetBrains", "AI IDE", "ThinkRail"]
summary: "JetBrains shipped an AI IDE, and getting it running showed me that the feature list isn't the interesting part: the source hardcodes readOnly: true, and when I typed into the editor the file hash didn't move. This is an IDE that won't let you change code."
cover:
    image: "/images/thinkrail/cover.png"
    alt: "JetBrains' New IDE, and the Editor Is Read-Only"
    relative: false
---

JetBrains released something new on August 19, 2026, called ThinkRail, and almost nobody in the Chinese tech circles has written about it.

I read the repo before I installed the local build. Reading the source turned up one thing that surprised me more than any of its features: the editor in this "IDE" is read-only.

Line 110 of `monacoSetup.ts` says `readOnly: true`, with a hint string to go with it. I opened the UI and tried typing into it. Not a single character went in, and the file hash didn't change.

This post is about that one thing: an IDE that won't let you edit code, and what exactly it's betting on.

## It Isn't the Next IntelliJ

The repo is at `JetBrains/thinkrail`, Apache-2.0, with a "JetBrains incubator project" badge on the README.

Their own description is one sentence: use pi to do vibe coding in this lightweight IDE, and it customizes itself around the way you work.

What's pi? A third-party minimal agent harness, package name `@earendil-works/pi-coding-agent`, from the Earendil folks. Its philosophy is "no features, just primitives". Sub-agents, plan mode, permission prompts, none of it is built in. You write your own extensions.

What ThinkRail does is wrap that command-line agent in a full interface: workspace, editor, diff, terminal, spec tree.

So what you're looking at is a company of a couple thousand people writing a shell in TypeScript to wrap an agent that somebody else wrote. That's a different road from the one IntelliJ took with its own kernel.

![](/images/thinkrail/fig1.png)

## Installing It Means a 218 MB File

It comes in two shapes: a desktop installer, and a self-contained command line build.

I downloaded the CLI version. A single 218 MB file with a Bun runtime stuffed inside, so you don't have to install node first. Running it looks like this:

`./thinkrail --no-open --port 24242 <your repo path>`

Open `localhost:24242` in a browser and you get the full interface. There's no account system of its own. Models come from pi's config.

I was all set to go into settings and paste a DeepSeek key, and then the Providers page opened with a line already sitting there: DeepSeek, env key `DEEPSEEK_API_KEY`, connected.

It had read the connection pi already had configured in my shell environment. Zero setup.

## I Gave It a Broken Repo

I prepared the test case on purpose: a small JS library that turns titles into URL slugs, with one of its two tests red because the implementation doesn't handle punctuation.

The landing page has three options: write specs, start work directly, or work inside the project directory. I clicked the first one, which is its signature flow.

I kept the prompt restrained: write out the specs as things stand, don't change any code yet.

After I hit create, the first thing it did was cut a real git worktree at `~/.thinkrail/worktrees/demo-repo/workspace-1`, on a branch called `write-slug-lib-specs`. I went back to the main repo to check. `main` was still on the same commit, not a hair moved.

This step isn't simulated. `git worktree list` shows two lines.

## Six Minutes Seventeen Seconds, Three Files

When it finished I looked at the numbers: 6 minutes 17 seconds, 26 tool calls, 15 steps, with three `spec_create`, three file reads, and three file edits. It used 5.3% of a one million token context and cost 0.138 dollars.

It didn't start writing right away. First `ls` to see the structure, read the README, read the tests, then it actually ran `node --test`, saw that red test, and only then started writing.

Three markdown files came out:

`goal-and-requirements.md` covers what the library is, who it's for, and that for now it only does "lowercase plus spaces to hyphens". `architecture.md` covers the single-module topology, with dependencies going only from tests to source. `src/SPEC.md` covers the `slugify` contract and its boundaries.

All three carry frontmatter, `id`, `type`, `status: draft` and `parent`, strung together into a graph. That's not a format I taught it. It's the fixed rule from the `spec-graph` skill pack it loads.

The most interesting part is how it handled that red test. It didn't write "strips punctuation" down as a capability that already works. Instead it opened a separate section in the module spec called "Known limitation" and spelled it out: the test expects `hello-world`, the function returns `hello,-world!`, punctuation left as is.

When it was done it ran `spec_validate` itself. It passed.

![](/images/thinkrail/fig2.png)

## What the Spec Graph Looks Like

The three files aren't documents sitting side by side. They're a tree with `parent` pointers.

The root is the goal document, the middle is architecture, the leaves are specs for individual modules. The right column in the UI is that tree. Click a node and the editor on the left renders the file, with Preview and Source views.

Their blog post "Spec-Driven Development" says the point is to let intent outlive the session. Close the chat and the code is still there, but the "why it's written this way" usually disappears along with the chat. Specs keep that part around, and the agent reads the whole graph in at the start of every run.

This time I only had it write, not edit. When it does edit, it reads the same graph.

![](/images/thinkrail/fig3.png)

## Then I Found the Read-Only Thing

After it finished I wanted to change a sentence myself, so I clicked into the Source view of `src/SPEC.md`, selected a line, and pressed a key.

Nothing. I assumed it was a focus problem, clicked a few times, then tried shoving text straight into the editable region with devtools. Same conclusion: input doesn't go in, and the file hash on disk is identical before and after.

So I went back to the source for the answer, and it's that `readOnly: true` from the top of this post. The UI even has a hint prepared, telling you to have the agent make the change.

Someone raised this on Hacker News, saying the chat panel is the lead actor in ThinkRail and the editor and file tree are supporting cast, that "you're not even allowed to edit files by hand".

I checked it, and it's true. In fairness though: the terminal works. There's a real PTY at the bottom, and nobody stops you from running `vim` in it. Only that Monaco panel is locked.

The problem is that this design swaps out the subject of "changing code". In an old IDE, you and the agent each own half of that. Here the division of labor is clear: the agent writes, you review.

![](/images/thinkrail/fig4.png)

## Why It Dares to Design It This Way

Because in its architecture, the IDE part was never the point.

The official architecture doc splits the system into three rings: the engine host, the protocol line, and the UI client.

The first ring runs in `packages/server`, using Bun to start an HTTP + WebSocket service with pi running in-process. The second ring is the versioned protocol in `packages/contracts`, the only coupling between the two ends. The third ring is `apps/web`, a React client that is mobile first.

"Mobile first" is not a figure of speech. The doc says so: the UI ships independently of the host, dials into a host over the network, and a phone can connect through Tailscale.

So from day one it never designed itself as a traditional IDE. It's a remote panel for an agent host, and the desktop build just happens to wrap it in Electrobun.

The read-only editor isn't a crippled editor. It's the natural result of that architecture: the UI is only the observer and the approver, and the thing that actually does the work is the agent inside the host.

![](/images/thinkrail/fig5.png)

## The Repo Itself

The codebase isn't small: 1,493 TypeScript files plus 349 TSX files, 354,000 lines.

The stack is new too: Bun runtime, React 19, Monaco 0.57 with shiki, xterm 6, Tailwind 4, TypeScript 6.0.3, Astro for the site. The desktop shell isn't Electron, it's Electrobun.

Community numbers: 7,484 stars, 450 forks, 365 open issues. The repo was created on July 8, 2026, and it got there in under four months. That's a fast pace.

Releases are dense too: 0.2.0 on October 18, 0.3.0 on October 29, 0.3.1 on November 1, and the nightlies are already at 0.4.0.

## Two Things That Don't Sound Good

First, telemetry is on by default. `analyticsEnabled` and `analyticsConsentConfirmed` are both true in `config.json`. That `--no-analytics` launch flag only turns off "additional usage data", not the basic stats, so you have to go into settings to kill those.

Second, JetBrains' AI strategy is a bit scattered right now. The two Hacker News threads together have over two hundred points and over two hundred comments, and the top voted one isn't praise, it's a complaint: Air and ThinkRail are both still half finished, while the AI Assistant that actually works well got pushed to the side.

I agree with that one. ThinkRail feels more like a bet, an experiment. JetBrains itself hangs the incubator sign on it.

## My Call

If your first reaction is "what use is an IDE that won't let you edit", then it really wasn't built for you.

It serves a different group: people already used to letting the agent do the work, whose main job is reviewing diffs and writing specs, moving forward one session after another. For them, being able to edit is actually a burden, because when you and the agent are fighting over the same file, the conflict can only be settled by self-discipline.

The answer it gives is physical isolation: one worktree and one branch per workspace, so the agent's mess stays on its own branch, and you merge only after you've reviewed it.

I think that idea is more thorough than "add an AI button to the editor". Its bet is that the IDE of the future is mainly for review, not for writing.

Whether the bet is right, check back in a year. As for this version, I'll use it for small jobs. I won't use it to replace IDEA.

When you write code these days, is that moment of changing a file still mostly your own hands, or has it already gone to the agent? If it's the latter, this read-only design probably won't bother you.
