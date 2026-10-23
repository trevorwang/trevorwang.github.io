---
title: "Does ADK for Kotlin Support iOS? I Took Its Gradle Metadata Apart to Check"
date: 2026-09-22T07:11:05+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "When I introduced ADK for Kotlin 1.0, I wrote that \"if iOS support lands later, that's one codebase reused across three platforms.\" That wording was slippery — placed next to \"KMP core,\" it reads way too easily as \"iOS should just work.\""
cover:
    image: "/images/adk-kmp-ios/cover.png"
    alt: "Does ADK for Kotlin Support iOS? I Took Its Gradle Metadata Apart to Check"
    relative: false
---

## First, a correction

When I introduced ADK for Kotlin 1.0, I wrote a line: "if iOS support lands later, that's one codebase reused across three platforms." That phrasing was slippery. Put it next to "KMP core" and it reads far too easily as "iOS should just work."

Then a reader asked: it's written in Kotlin Multiplatform, so doesn't iOS already work?

I should not have glossed over that. So this time I went and checked myself, and not by reading what the docs say. I took the build artifacts apart.

The conclusion, up front: **no. And this is not something one extra target line fixes.**

## There is a hard way to check what a KMP library actually supports

Method first, because the method is more useful than this particular conclusion.

When a KMP library publishes, it generates a **Gradle module metadata** file (the `.module` file) listing every platform variant the library actually compiled. It is a build system artifact, not marketing copy someone wrote. A library can claim it "supports multiple platforms," but if the metadata has no variant for one, the build simply cannot resolve it and compilation fails outright.

So to learn a library's real platform coverage, one command is enough:

```bash
# Generic recipe: swap the package path in for group
curl -sL "https://repo1.maven.org/maven2/com/google/adk/google-adk-kotlin-core/1.1.0/google-adk-kotlin-core-1.1.0.module" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
vs = d.get('variants', [])
plats = sorted({v['attributes'].get('org.jetbrains.kotlin.platform.type', '?') for v in vs})
names = [v['name'] for v in vs]
ios = [n for n in names if 'ios' in n.lower()]
print('platform.type :', ', '.join(plats))
print('含 ios 的变体 :', len(ios), '个')
for n in ios[:4]: print('   -', n)
"
```

Run it against ADK for Kotlin and the output is:

```
platform.type : androidJvm, common, jvm
含 ios 的变体 : 0 个
```

## How to read that result

One thing is easy to misread. I nearly confused myself the first time: **the iOS target does not show up in `platform.type`.**

Kotlin/Native is uniformly tagged `native` under `platform.type`. Which Apple platform it is shows up in the **variant name**. So the correct read is both layers together:

- `platform.type` contains `native` → the library compiles Kotlin/Native targets
- **variant names include `iosArm64` / `iosSimulatorArm64` / `iosX64`** → only then does iOS actually work

Compare against a library that definitely supports iOS, Ktor say:

```
platform.type : common, js, jvm, native, wasm
含 ios 的变体 : 9 个
   - iosArm64ApiElements-published
   - iosArm64SourcesElements-published
   - iosSimulatorArm64ApiElements-published
   ...

```

64 variants, 9 with ios. That is what "actually supports iOS" looks like.

ADK has 8 variants. `platform.type` is only androidJvm / common / jvm, and **variants with ios number zero**.

I checked every version while I was at it, including the latest 1.1.0. No iOS anywhere.

The source layout agrees. Under `core/src/` the source sets are:

```
androidMain
commonJvmAndroidMain      ← note the name
commonMain
jvmMain
(no iosMain)
```

And the target declarations in `core/build.gradle.kts` are just `jvm()` plus Android. Not one line of iOS.

Under the `com.google.adk` group on Maven Central, **the number of iOS-related artifacts is zero.**

## Three walls, each harder than the last

Knowing there is no iOS is not enough. It is worth seeing why, because the answer decides how much "just do it yourself" costs.

### Wall one: they deliberately split out a layer called `commonJvmAndroidMain`

The source set name is the answer. It is the part JVM and Android share. It `dependsOn(commonMain)` and contains:

| Dependency | Status on iOS |
|---|---|
| `kxml2` (XML parsing) | No Kotlin/Native version; swap libraries or write an actual |
| `snakeyaml` (YAML) | Same |
| `opentelemetry-api` (Java telemetry) | iOS would need OpenTelemetry's Swift implementation; the whole layer has to become pluggable |
| `reactive-streams` | **A JVM-only spec. Apple platforms have no counterpart at all** |

`reactive-streams` is the key signal. It is the JVM world's reactive streams spec, and nothing like it exists on Apple platforms. ADK's Java interop signatures are built on top of it. The source comment puts it as "the api-vs-implementation question for the Java-interop signatures".

The authors know perfectly well the code has two layers: one that is genuinely cross-platform common logic, and one that is "shared by JVM and Android." **The distinction is deliberate, because right now they only want to support those two platforms.**

### Wall two: `jvmMain` is all JVM-only too

`gson`, `google-cloud-storage`, `mcp` (the Java SDK for MCP), `ktor`. Neither Google Cloud Storage nor the MCP Java SDK has an iOS version, so session persistence and cloud storage would need a separate iOS implementation.

### Wall three (the fatal one): `google-genai-kotlin` does not support iOS either

`commonMain` has this line among its dependencies:

```kotlin
commonMain {
  api(libs.google.genai.kotlin)   // com.google.genai:google-genai-kotlin:1.0.0
}
```

I ran the same method on it:

```
Platform variants: androidJvm, common, jvm
```

**Google's own Kotlin SDK for generative AI has no iOS target either.**

And it is an `api` dependency, not `implementation`. That means it sits on every consumer's compile classpath. There is no way around it.

So the truth is: **to get ADK for Kotlin running on iOS, you first have to port Google's GenAI Kotlin SDK to iOS.** That is a whole other SDK's worth of work, and it lives in Google's hands, outside anything you can change.

## So what does "compile it myself with iOS support" actually mean

Not a fork plus three target lines. Roughly:

| What it takes | Size |
|---|---|
| Add iOS targets plus actual implementations per layer (XML, YAML, telemetry, persistence, networking) | Medium to large |
| Replace `reactive-streams` with Flow, redesign the API (breaks the Java interop promise) | Medium |
| Port google-genai-kotlin to iOS, or bypass it and hit REST directly | **Large** |

The last two are not in your repo. Especially the third: that means moving someone else's code.

## 1.1.0's direction makes this clearer, not murkier

ADK for Kotlin 1.1.0 shipped September 15 (1.0 went GA September 7, 1.0.1 on September 10 — fast cadence). Look at what it adds:

- `AppFunctionsToolset`, wiring into Android's AppFunctions
- An Android AppFunctions conversion layer
- Injects the app's AppFunctions descriptions into model instructions
- A `BaseFutureSkillSource` base class for Java, plus Java examples of long-running tools
- The `Plugin.onRunError` callback

The fixes land in the same area: unknown tool names no longer kill the whole call, session recovery is fixed for workflow agents, and a turn survives when an MCP server rejects a request.

Look at where the weight sits. The most substantial new capability in 1.1.0 is the **Android AppFunctions** line — letting an agent call functions an Android app exposes.

Which tells you the iOS gap is not an oversight. It is a **strategic choice**. Google's energy is on the Android side: Gemini Nano, ML Kit, LiteRT-LM, Firebase AI, and now AppFunctions, all laid down for Android. ADK for Kotlin's "Kotlin Multiplatform" is currently shorthand for "Kotlin Multiplatform (Android + JVM)." That common layer exists for those two platforms to share, not as groundwork for iOS.

## So what do you do if you build for iOS

Three paths, in my order of preference.

**Path A: run the agent on the server, keep iOS as a client (recommended)**

ADK for Kotlin's **JVM leg is fully usable.** That is one of its main battlegrounds, not a crippled build.

So run the agent logic on a server JVM and have iOS call it over HTTP. The ADK ecosystem even has the piece for this: the A2A (Agent-to-Agent) protocol, which is exactly what the `google-adk-kotlin-a2a` module is for (it is also JVM-only, but servers are JVM anyway).

The cost is losing on-device capability. Privacy, offline and low latency no longer all fit.
The upside is you can ship today with zero porting risk.

For most products that trade is worth it: your agent already calls a model over the network, so it was never purely offline.

**Path B: write your own "ADK-lite for iOS"**

Take only the design ideas from commonMain — orchestration, sessions, memory, tool schemas — and rebuild them on iOS out of parts the KMP ecosystem already has:

- Ktor for networking (KMP supports iOS natively)
- Call the Gemini REST API directly, bypassing `google-genai-kotlin`
- SQLDelight for persistence

But then you are not "using ADK." You are reimplementing it from its design. Business logic can be shared. The framework gets rewritten.

**Path C: wait, or push**

1.0 to 1.1.0 was 8 days. Iteration is not slow. If iOS is on the roadmap, waiting pays off.

One thing I did while I was at it: I opened an issue in the repo to get this settled —

> https://github.com/google/adk-kotlin/issues/504

I asked three things: is iOS on the roadmap, what is the actual blocker (I suspect it is the GenAI Kotlin SDK missing a native target, not ADK itself), and what architecture does the team recommend today.

Why I think the issue is worth something: the repo only has a handful of issues total, and one feature request filed last month got a `need attention` label within two days. The team is watching. If you need iOS, go add a thumbs up. Demand signals beat everything else.

## The method is worth more than the conclusion

Back to the method from the top.

Next time you evaluate any KMP library — ADK, or some other "cross-platform" framework — do not just read the README and release notes. The README will say things like "runs on all supported platforms," which looks comprehensive, but who defines "supported" and how far it goes is usually left out.

**Go read the `.module` file.** It lists what the build system can actually resolve:

- `iosArm64` / `iosSimulatorArm64` present → iOS really works
- `js` / `wasmJs` present → web really works
- only `common` / `jvm` / `androidJvm` → JVM and Android only

One curl saves you two weeks of refactoring.

That is the biggest cognitive trap in KMP these last couple of years: **"built on KMP" and "works on all KMP platforms" are different claims.** The first is about how the code is organized. The second is about what you actually get. ADK for Kotlin is a good example — the core really is a carefully designed cross-platform architecture, and right now it chooses to keep the iOS door shut.

---

*How I verified: read the Gradle module metadata for every version of `com.google.adk:google-adk-kotlin-core` (including 1.1.0) on Maven Central, the `core/build.gradle.kts` and `core/src/` source set layout, the platform variants of `com.google.genai:google-genai-kotlin`, and cross-checked the release notes for ADK for Kotlin v1.0.0 / v1.0.1 / v1.1.0.*
