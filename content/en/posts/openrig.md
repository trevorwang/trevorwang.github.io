---
title: "One YAML, One Agent Team"
date: 2026-10-03T22:24:22+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent"]
summary: "It is two in the morning, the machine has started swapping, you have hit later on the update prompt three times, and you still will not reboot."
cover:
    image: "/images/openrig/cover.png"
    alt: "One YAML, One Agent Team"
    relative: false
---

## Before you reboot, you cannot count how many agents are open

It is two in the morning, the machine has started swapping, you have hit later on the update prompt three times, and you still will not reboot.

Because you have five agents running: one refactoring, one reviewing a PR it submitted an hour ago, and two digging through other repos. They are all busy. You are just not entirely sure which terminal window is which.

That anxiety opens mvschwarz's April post, Why I Built OpenRig. His fix is blunt: write the whole agent team into one YAML file, `rig up` to bring the lot online, and restore it exactly as it was after a reboot.

I installed OpenRig on my own machine and took it apart. The most useful line first: on first run it modifies your Claude Code and Codex config, so do not casually `npm i -g` it on your main machine.

## The layer it manages is called topology

Terms first. Things like Claude Code and Codex CLI are called harnesses, and a harness wraps a model. OpenRig does not replace them, it wraps them.

> A harness wraps a model. A rig wraps your harnesses. Define your agent team in YAML, boot it with one command. Claude Code and Codex in the same rig, managed as one system.

The author calls this distributed context management. What people usually mean by context engineering is watching one agent's window: when to compact, when to hand off to a subagent. He inverts it: keep each agent in its own narrow, focused context and coordinate through formation, instead of cramming everything into one window.

The whole team shares one CULTURE.md. The author calls it the group's constitution, the equivalent of SOUL.md in OpenClaw that defines one agent's personality, except scoped to the entire team.

For this CLI specifically, I counted the subcommands in `--help`: 86. Beyond the guessable `up`, `down`, `ps`, `tui`, there are queues, a workflow runtime, watchdogs, cost telemetry, snapshot restore, and claiming stray tmux sessions back into formation. This is less a tmux wrapper and more a small job system.

## First step in testing: get through the environment gate

OpenRig requires Node 22 or 24, plus tmux. I installed the package into a temp directory instead of globally:

```bash
npm i @openrig/cli          # 0.6.4, 110 dependency packages
./node_modules/.bin/rig preflight
```

The preflight result was blunt:

```text
✓ node_version
✗ tmux: tmux was not found in PATH.
  Why: OpenRig uses tmux to create and control agent sessions.
  Fix: Install tmux (brew install tmux on macOS, apt install tmux on Debian/Ubuntu).
✓ writable_home
✓ port_available

1 check(s) failed.
```

Three of four passed, blocked on tmux. One trap worth mentioning: this machine runs node 25, and OpenRig only supports 22 and 24 — npm 12 itself warns that node 25 is unsupported. I got it running by leaning on the node@22 already in Homebrew. So do not skip this step before you start; let it tell you what is missing first.

## Before installing, see which of your files it will touch

`rig setup` is the big environment prep step: install tmux, install cmux, install the agent CLIs, configure auth. I did not want it acting for real, so I ran `--dry-run` first.

It did not hedge. It listed the files it would touch:

![](/images/openrig/setup-dry-run.png)

```text
[global]  claude-code  ~/.claude.json                pre-trust managed workspace, mark onboarding done
[project] claude-code  .claude/settings.local.json   write statusLine and acceptEdits floor snippet
[project] claude-code  .mcp.json                     write the selected MCP resource snippet
[global]  codex        ~/.codex/config.toml          pre-trust + config snippet
[global]  cmux         ~/.config/cmux/settings.json  switch cmux's socket control into compatibility mode
```

(The list above is my translation; the original is in English.)

Two things surprised me. First, it states plainly that it bakes no permission policy — the words are OpenRig bakes NO allow/ask/deny permission policy, with native harness permissions as the control plane. Second, when it comes to skipping confirmations, it hands the decision back to you, and your agent asks once before the team actually comes up:

> Allow your agents to run OpenRig commands without repeated permission prompts?
> Yes — recommended / No — keep prompts. No answer leaves settings unchanged too.

Pick No or simply do not answer, and it writes nothing. Another line from the docs worth bookmarking: recording a permission policy is a note of intent, not a gate, and `rig up` runs without it.

So does `--dry-run` really not write? I took SHA256 hashes of those files beforehand; after the run they were identical, the skills directory still had 93 entries, and `~/.openrig` was never created. It is a genuine dry run.

## One team is one YAML

That dry-run list only covers the config layer. The actual team is defined in YAML, and the package ships 19 rig definitions, 16 agent definitions and 6 workflow definitions.

The smallest one looks like this:

```yaml
version: "0.2"
name: first-project
culture_file: CULTURE.md
pods:
  - id: dev
    label: Project
    members:
      - id: owner
        agent_ref: "local:../../../agents/development/implementer"
        runtime: codex
        model: gpt-6-astra
        cwd: "."
      - id: check
        agent_ref: "local:../../../agents/development/qa"
        runtime: codex
        model: gpt-6-astra
        cwd: "."
    edges:
      - kind: delegates_to
        from: owner
        to: check
```

Two seats in one pod, owner and check, the second an independent reviewer, joined by a `delegates_to` edge. That edge is not decoration; it has runtime meaning, it is the communication path between seats. The same definition also comes in a Claude version and a mixed version, first-project-claude and first-project-mixed.

One neat touch: the first time it starts the daemon, it brings up a resident rig called kernel. Three pods. advisor commands through Claude Code; operator holds one terminal seat for the human plus a Codex seat; queue has a worker that classifies input streams into queues.

Between the human seat and the agent seats run two edges, `collaborates_with` and `escalates_to`, meaning what the agent cannot do escalates to you.

![](/images/openrig/kernel-topology.png)

## What is actually in a seat

One level deeper: a seat is an agent definition. The implementer role on the dev side looks like this:

```yaml
name: implementer
defaults:
  runtime: claude-code
profiles:
  default:
    uses:
      skills: [development-team, test-driven-development, systematic-debugging, verification-before-completion]
      plugins: [shared:openrig-core]
      runtime_resources: [shared:claude-default-settings, shared:claude-default-mcp, shared:codex-default-config, shared:claude-activity-hooks]
startup:
  files:
    - path: guidance/role.md
      delivery_hint: send_text
```

In plain terms: which runtime this seat uses, which skills it carries, which plugins and resources it mounts, and what role description gets sent in at startup. A seat is not "open a terminal and explain as you go"; it is a reusable, versionable job description. Same for the workflow layer: roles declares the roles, steps run in declared order, and the example just routes one package through the plan, build and prove seats in sequence.

## Where it stands now

OpenRig is Apache-2.0. The first release went out April 6; by 0.6.4 on October 2 it had shipped 54 versions, 4.2k stars, 38 contributors, and 4,560 npm downloads last week. The version is still 0.6, with spec versions RigSpec 0.2 and AgentSpec 1.0.

The author compares it with Claude Managed Agents, which Anthropic shipped on April 8 this year: cloud hosted, Claude only, billed per session hour. His side runs locally, mixes Claude and Codex, and spends your own subscription quota. His analogy is Terraform and AWS — one manages resources in the cloud, the other manages local formation.

> Terraform doesn't compete with AWS. OpenRig doesn't compete with Managed Agents.

Writing my team into one file? I am willing. What actually stops people is not the syntax, it is accepting one thing up front: to manage your agents, it first edits your agent config, and does it globally. Do you take that trade, or do you figure five windows in tmux by hand is fine too?
