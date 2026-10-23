---
title: "AGENTS.md Is Here. So How Should a KMP Project Write This Manual?"
date: 2026-09-19T07:19:11+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "AI Agent"]
summary: "Starting with yesterday's news: Claude Code 2.1.277 now supports AGENTS.md (September 18)."
cover:
    image: "/images/agents-md-kmp/cover.png"
    alt: "AGENTS.md Is Here. So How Should a KMP Project Write This Manual?"
    relative: false
---

## A standoff that lasted nearly a year ended yesterday

Starting with yesterday's news: **Claude Code 2.1.277 now supports AGENTS.md** (September 18).

The Register's headline was blunt: **"Anthropic decides to support OpenAI's markdown instruction spec."**

If you don't live inside AI coding tools every day, you might think: it's a config file name. Who cares?

**It matters. Because this whole standoff started with a hard line the Shopify CEO said in public.**

Rewind to August 26. Shopify CEO **Tobi Lütke** posted on X:

> I am considering banning Claude Code at Shopify **until they are willing to read AGENTS.md, .agents/skills and similar files.**

An e-commerce giant threatening to ban a mainstream AI coding tool company-wide — **over nothing more than its refusal to read one markdown file.**

And the reason he gave is, to me, worth more than the news itself:

> Both AGENTS.md and CLAUDE.md apply recursively down the directory tree. When thousands of developers maintain one monorepo, **some directory will inevitably be missing a file, and the result is that some developers end up with an agent that has had a chunk of its brain cut out.**
>
> We solved it with automation, but that is a **stupid complexity tax**, and we should not have to pay it at all.

**"An agent with a chunk of its brain cut out"** — if you have ever maintained a large repo, that one sinks in.

## First, what AGENTS.md actually is

The official definition is one sentence:

> **Think of AGENTS.md as a README for agents.**

**A README for agents.**

A README is for humans: how to get up and running fast, what the project does. AGENTS.md holds what the agent needs but does not belong in a README: **build steps, how to run tests, code conventions, a few hard rules.**

Its standing today is higher than a lot of people realize:

- Adopted by **60,000+ open source projects**
- Started as a collaboration between several: **OpenAI Codex, Amp, Google's Jules, Cursor, Factory**
- Now stewarded by the **Agentic AI Foundation under the Linux Foundation**

The minimal example from the official docs looks like this:

```markdown
# AGENTS.md
## Setup commands
- Install deps: `pnpm install`
- Start dev server: `pnpm dev`
- Run tests: `pnpm test`

## Code style
- TypeScript strict mode
- Single quotes, no semicolons
```

**That is all it is.** No schema, no special syntax. Just markdown.

What makes it actually useful: **one file works across multiple agents**. Codex, Cursor, Jules, Copilot and Claude Code all read it, so you are not maintaining a config per tool.

> Side note: Claude Code insisted on its own `CLAUDE.md` for a long time. Developers started asking in August 2025, the issue collected nearly **5,000 GitHub reactions**, and it was finally closed as won't fix. **It took Shopify threatening a ban to get support on September 18.**

## Why one file escalates into a platform engineering problem

Plenty of people will think: it is just getting Claude Code to read one more file. Why would Shopify escalate to a company-wide ban?

**Because in a monorepo with thousands of people, one file format discrepancy turns into chaos you cannot control.**

The mechanism Lütke described is worth copying down:

```
monorepo/
├── AGENTS.md          ← global rules (tech stack, commit conventions)
├── backend/
│   └── AGENTS.md      ← backend-specific (do not touch db/migrations)
├── mobile/
│   └── AGENTS.md      ← mobile-specific (packaging, signing)
└── infra/
    └── AGENTS.md      ← infrastructure-specific
```

**AGENTS.md applies nearest-first** — the agent reads the closest one in the current directory and inherits the rules above it.

That is where the trouble starts:

- Say the team uses both Cursor (reads AGENTS.md) and Claude Code (only reads CLAUDE.md)
- Someone writes `AGENTS.md` under `mobile/`, and nobody remembers to write a `CLAUDE.md` too
- Developer on Cursor: the agent knows to run the signing script before packaging
- Developer on Claude Code: the agent **does not**

**Same code, same task, two different results.** And no alert will ever catch it — **you just occasionally notice that a colleague's AI is "especially dumb."**

**That is the complexity tax Lütke was talking about**: not that it cannot be solved, but that **the problem should not exist in the first place.**

## Down to practice: writing one for a KMP + AI chat client

Okay, news done. On to the actual subject.

Say we are building a **Kotlin Multiplatform + Compose Multiplatform AI chat client** — one Kotlin codebase running on Android and iOS.

A project like this has its own quirks when writing AGENTS.md: **in one repo, three kinds of code follow completely different rules.**

Start with the directory tree:

```
ai-chat/
├── AGENTS.md                      # root: global hard rules + common commands
├── settings.gradle.kts
├── gradle/
│   └── libs.versions.toml         # version catalog (single source for dependency versions)
├── shared/                        # shared business logic (no UI)
│   ├── AGENTS.md                  # rules for this layer
│   ├── build.gradle.kts
│   └── src/
│       ├── commonMain/kotlin/     # ← shared across platforms
│       ├── androidMain/kotlin/    # ← Android implementation
│       ├── iosMain/kotlin/        # ← iOS implementation
│       └── commonTest/kotlin/
├── composeApp/                    # Compose Multiplatform UI
│   ├── AGENTS.md
│   └── src/
│       ├── commonMain/
│       ├── androidMain/
│       └── iosMain/
├── androidApp/                    # Android shell project
│   ├── AGENTS.md
│   └── src/main/AndroidManifest.xml
└── iosApp/                        # iOS shell project (Xcode)
    ├── AGENTS.md
    └── iosApp.xcodeproj
```

**Note: the structure itself is the first layer of information for the agent.** If the agent cannot tell `shared/` from `composeApp/`, it will write UI code into the logic layer.

### Root AGENTS.md: only global hard rules and common commands

```markdown
# AGENTS.md

An AI chat client built with KMP + Compose Multiplatform.
One Kotlin codebase, compiled to Android / iOS.

## Project structure
- `shared/`     — shared business logic, **contains no UI**
- `composeApp/` — Compose Multiplatform UI (all platforms share one set of screens)
- `androidApp/` — Android shell (Activity / Application / manifest)
- `iosApp/`     — iOS shell (Xcode project / SwiftUI entry point)

## Build and test
- Full build:             `./gradlew build`
- Shared module compile:  `./gradlew :shared:compileKotlinMetadata`
- Install to Android:     `./gradlew :androidApp:installDebug`
- Run all tests:          `./gradlew allTests`
- Run one class:          `./gradlew :shared:allTests --tests "*ChatRepository*"`
- iOS framework:          `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`

## Hard rules (violating these breaks compilation on every target)
1. `java.*`, `android.*`, `platform.*` are **forbidden** in `commonMain`
2. All platform differences go through `expect` / `actual`, never `if (isAndroid)`
3. **Do not copy the logic for iOS** — that throws away the point of using KMP
4. New dependencies go into `gradle/libs.versions.toml` first; modules reference only the alias, **never hardcode a version**
5. After changing `commonMain`, it must still compile for both Android and iOS
```

**A few things about how this is written that I think matter a lot:**

**① Commands must be copy-pasteable, not descriptions.**
Write `./gradlew :shared:compileKotlinMetadata`, not "compile the shared module." **Give the agent something it can run directly and it does not have to guess.**

**② Hard rules need their consequences spelled out.**
"No `java.*` in commonMain" is a rule. Add "**it breaks compilation on every target**" and the agent understands it is a hard constraint, not a style preference.

**③ Kill ambiguity with a single source of truth.**
"Versions live only in `libs.versions.toml`" — that one line stops the agent from hardcoding versions all over the place.

### shared/AGENTS.md: only what is specific to this layer

**A subdirectory AGENTS.md should not repeat the root file.** It should hold only what you need to know once you are in that directory:

```markdown
# shared/ — shared business logic

## What each source set is for
- `commonMain/` — platform-agnostic logic. Need a platform capability, declare it here with `expect` first
- `androidMain/` — the Android implementation of each `expect` (Android SDK / JVM APIs)
- `iosMain/`     — the iOS implementation of each `expect` (Kotlin/Native + Foundation)
- **Every new `expect` needs both `actual` implementations, in androidMain and iosMain**

## Directory conventions
- `data/`      — repositories, data sources, DTOs
- `domain/`    — domain models, use cases
- `platform/`  — platform abstractions behind expect/actual
                 (KeyValueStore, HttpEngine, AppInfo…)

## Networking (the core of an AI chat app)
- Use **Ktor** everywhere. No OkHttp or URLSession directly in the shared layer
- Streaming responses always return `Flow<ChatChunk>`. Do not manage UI state in shared
- Timeouts, retries and error mapping all live in `data/remote/`

## Commands
- Tests:        `./gradlew :shared:allTests`
- iOS framework: `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`
```

**See what is in there? This layer holds purely KMP-specific knowledge:**

- the pairing rules for `expect`/`actual` (the easiest place to screw up in KMP)
- why networking has to be Ktor (OkHttp does not exist on iOS)
- why streaming uses `Flow` (the defining feature of an AI chat app)

**None of this goes in a general README, and the agent trips on it every single time.**

### The shell projects: each one says what not to touch

**androidApp/AGENTS.md**:

```markdown
# androidApp/ — Android shell

## What goes here
- Activities, Application, AndroidManifest.xml, res/ resources
- **Business logic always goes in shared/, UI always goes in composeApp/**

## Common commands
- Install:  `./gradlew :androidApp:installDebug`
- Logs:     `adb logcat -s AiChat:V`

## Watch out
- New permissions require an AndroidManifest.xml change and **an update to this file**
- Before changing minSdk, check API compatibility for what shared/ uses
```

**iosApp/AGENTS.md**:

```markdown
# iosApp/ — iOS shell (Xcode project)

## Important: this is not built with Gradle
- This is an Xcode project. **Do not try to build it with gradlew**
- After changing code in shared/, run this first:
    `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`
  so Xcode picks up the new framework, then Run from Xcode

## The Swift side is glue only
- Lifecycle, permission requests, navigation containers
- Business logic **always goes back to shared**. Never rewrite it in Swift

## Common traps
- Missing permission description in Info.plist: only fails at runtime on the simulator, Xcode says nothing
- After changing framework exports, Clean Build Folder
- Kotlin sealed classes must be exhaustively switched on in Swift (write every branch, no default)
```

**Why three separate files?**

Because **what to do and what not to do flip completely from one directory to the next:**

- In `shared/`, you write: do not touch platform APIs
- In `androidApp/`, you write: do not put business logic here
- In `iosApp/`, you write: do not build this directory with Gradle

**One repo, rules that contradict each other.** A single root AGENTS.md cannot hold this — **that is exactly why nearest-first exists.**

### Side note: mixed tools on the team

AGENTS.md is only a convention, and different tools still read it differently. The docs give some config pointers:

- **Aider**: add one line to `.aider.conf.yml` — `read: AGENTS.md`
- **Gemini CLI**: configure `{ "context": { "fileName": "AGENTS.md" } }` in `.gemini/settings.json`
- **Migrating from the old name**: `mv AGENT.md AGENTS.md && ln -s AGENTS.md AGENT.md` (the symlink keeps compatibility)

In a KMP project there is also the case of **uneven tool versions** — someone's Claude Code is still below 2.1.277. You can drop a `CLAUDE.md -> AGENTS.md` symlink in the key directories as a stopgap.

**But remember what Lütke said: that is paying the complexity tax.**

**If you can upgrade everyone's tools, do not maintain two files.** Symlinks are a transition, not the destination.

## A few traps I have hit

**① Do not turn AGENTS.md into a translated README.**
A README is a project intro for humans. AGENTS.md is an operating manual for the agent. The difference: **a README says what, AGENTS.md says how, and it has to be executable.**

**② Do not write should, recommend, try to.**
How an agent handles vague wording is unpredictable. Either it is a hard rule (with the consequence stated) or leave it out.

**③ Do not let it go stale.**
**A stale command is worse than no command** — the agent runs it and gets stuck on a task that no longer exists. When you change a build script, update AGENTS.md in the same pass. Make it a habit.

**④ Do not overstuff it.**
The official minimal example is a dozen lines and it is enough. Context is a budget, and **every line you write for the agent takes space away from understanding the code.**

**⑤ Draw the directory tree.**
Best return on effort of anything here. **One accurate tree saves the agent ten `ls` calls.**

## What I think

**First, the AGENTS.md outcome is really platform-neutral beating tool-locked.**

Anthropic held out for nearly a year and then supported it anyway. The reason is simple: **when 60,000 projects use one format, not supporting it means taxing your users.**

That complexity tax line from Shopify hits the real pain every large repo has.

**Second, for KMP projects AGENTS.md is probably worth more than for an ordinary project.**

Because KMP's traps are **unusually directory-specific**:

- Does this code belong in `commonMain` or `iosMain`? Completely different rules
- Does this API exist on every platform?
- Do I need to relink the iOS framework after this change?

**Saying all of that out loud in every conversation is expensive. Written into directory-level AGENTS.md files, it becomes a one-time investment.**

**Third, the long-term significance may be bigger than it looks.**

As AI coding tools move from completing code to finishing tasks on their own, **does the repo have a file that states the rules** starts to look a lot like does this project have automated tests — **a marker of engineering maturity.**

There is a side benefit too: **writing AGENTS.md is itself an exercise in cleaning up team conventions.** It forces you to write down the implicit rules that only old-timers know — **and that may be worth more than making the AI a little smarter.**

---

I would like to ask you:

**Does your project have an AGENTS.md?** If not, which part do you think is hardest to write: the build commands, or the conventions that live only in people's heads?

**Or what trap have you hit in a KMP project that documentation later fixed?** Tell me in the comments and I can turn it into the next post.

---

*Sources: the AGENTS.md spec (agents.md), The Register (9/18), 36Kr / CSDN coverage of the Shopify–Anthropic dispute (8/26), the Claude Code 2.1.277 release notes.*
