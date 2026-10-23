---
title: "The Laziest Senior Dev on the Team Is Now Writing Rules for AI Agents"
date: 2026-10-05T07:57:08+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "You ask the agent to add a date picker. It installs a calendar library, writes a wrapper component, adds a stylesheet, and then starts discussing time zones with you."
cover:
    image: "/images/lazy-agent/cover.png"
    alt: "The Laziest Senior Dev on the Team Is Now Writing Rules for AI Agents"
    relative: false
---

## Start with that familiar guy

You ask the agent to add a date picker. It installs a calendar library, writes a wrapper component, adds a stylesheet, and then starts discussing time zones with you.

Which is when you think of that person in the office. Ponytail, oval glasses, been at the company longer than version control. You hand him fifty lines of code, he glances at it, says nothing, and comes back with one line.

What the ponytail project does is stuff him into your agent. 155K stars, MIT license, and the first line of the README reads: he says nothing, he writes one line, it works.

## Agents are born adding things

An agent's default move is addition. Leave it alone and it outfits every request with something new: a new dependency, a new abstraction, new boilerplate. You cannot blame it entirely; that is what the engineering practices in its training data look like.

The problem is it does not care about your repo, and it does not care about your bill. That confirmation dialog above — all you wanted was "are you sure you want to delete this" — and browsers have had a native `dialog` since 2022, with focus management, Esc to close and a backdrop thrown in free.

That "write less code" camp on the trending lists these past two days is talking about exactly this.

## Its entire estate is one 2.5KB rule file

The core of ponytail is not code. It is a 2,492-byte rules file. Inside is a ladder: start at the first rung and climb, stopping wherever one holds:

```text
1. Does this need to be built at all? (YAGNI)
2. Does it already exist in this codebase?
3. Does the standard library already do this?
4. Does a native platform feature cover it?
5. Does an already-installed dependency solve it?
6. Can this be one line?
7. Only then: write the minimum code that works.
```

> In other words: ① does this thing even need to exist? ② is there one already in the repo? ③ can the standard library do it? ④ does a native platform feature cover it? ⑤ can something already installed solve it? ⑥ can it be one line? ⑦ only now do you write "the minimum code that runs."

There is one more line sitting above the ladder: read the task and the code it touches first, walk the real flow end to end, then start climbing. Reverse the order and it is not laziness, it is using efficiency as an excuse.

## I got it running

These rules are spread wide. Claude Code, Codex, Cursor, Windsurf, Copilot and Hermes each get an adapter file. I picked the cleanest route, since it ships its own MCP server:

```bash
cd ponytail-mcp && npm install && node index.js   # speaks MCP over stdio
```

Walking through a real handshake, this is what it handed me:

![](/images/lazy-agent/mcp-handshake.png)

The three intensity levels (lite / full / ultra) differ by only a few dozen characters in body text. Which means intensity adjusts tone and strictness, not prompt length. The rules are in there at every level, around 5,200 characters.

The README also mentions an easily missed detail: over MCP, the rules are pulled on demand. If you want them active every single turn, you have to go the hook or plugin route. Do not assume MCP means it has taken over. It depends which end you connected.

## The numbers it saves, and what it does not save

The figures in the README are real runs. Take a real open source repo as the target, have the same agent do twelve feature tasks with and without it, and compare the git diff it leaves behind: 54% less code, 22% fewer tokens, 20% lower cost, 27% less time, all safety checks passing.

A few concrete examples, same model, same prompt:

- Search box debounce: 116 lines → 10 lines
- CSV sum: 20 lines (with pandas) → 3 lines (csv stdlib)
- Delete confirmation dialog: 1 dependency + 30 lines → 0 dependencies + 8 lines (native dialog)

Then the column I think matters most: the control group. Tell it only "write it as short as possible" and you get 33% less code, but safety checks drop to 95%. Say less without rules and you get 20% less code, while tokens go up 7% and cost up 3%.

So writing less is not worth anything by itself. What is worth something is the other list it writes down at the same time, the places where laziness is not allowed: understanding the problem, input validation at trust boundaries, error handling that prevents data loss, security, accessibility, calibration on real hardware. Plus one more: non-trivial logic needs a runnable check, the kind that fails the moment the logic breaks.

One more thing that is rare: the README walks its own numbers down. The earlier single-round benchmark reported 80% to 94%, and a later note states that against a fairer baseline, that was the ceiling for a single task, not the average. The mean is 54%.

## The other path is addition

Also trending next to it these past two days is ECC, with more stars, going in exactly the opposite direction. The repo has 4,212 files, 1,027 SKILL.md files and a hundred thousand characters of README, and claims to offer 293 skills and 68 agents.

It does not teach the agent to write less. It packs skills, instincts (with confidence scores, learned from real sessions), memory, security scanning and a research-first workflow into the harness. The slogan is honest: optimize the context window, persist everything else.

![](/images/lazy-agent/two-camps.png)

The disagreement between the two camps is not about laziness. It is about where you spend the budget. One spends it producing less code, the other spends it remembering more things. The whole ponytail repo is 1.78MB. ECC is 56MB.

## How it lands on my own machine

The agent client I normally use is DeepSeek Harness (dsh), whose slogan is Everything is a Plugin. This is how I mount MCP in it:

```yaml
- insert:
    - id: memory-memorix
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: memorix
        transport: stdio
        command: memorix
        args: [serve, --mode, lite]
```

ponytail's MCP server speaks the same stdio protocol, so wiring it in means swapping out command and args. Think one thing through before you do: once something like this goes in through a plugin or hook, it enters your context every single turn, and you pay the tokens.

## Where I land

Writing it as a checklist you can hold up against your own setup:

- Is it saving "writing" or "rework"? Less code does not mean fewer potholes, and rework rate is not in its metrics.
- Do the rules enter every turn? MCP pulls on demand, plugins and hooks inject per turn. The two have different bills.
- What in your project must not be cut? Write it down. It does far more than saying "don't be verbose."
- The addition path is not wrong either, but do the context math first.

Back to that colleague with the ponytail. What earns him respect is not that he writes less. It is that he knows every extra line will need a maintainer someday. Putting rules in front of an agent is the same: the point was never to make it type less, it is to make it know what it must not touch.

What is the most unnecessary thing an agent has shoved into your project lately? Tell me in the comments.

References:
- https://github.com/DietrichGebert/ponytail
- https://github.com/affaan-m/ECC
