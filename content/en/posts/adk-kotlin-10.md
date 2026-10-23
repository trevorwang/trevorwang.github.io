---
title: "Kotlin Developers Can Finally Write Production-Grade AI Agents: A Deep Dive into Google ADK for Kotlin 1.0"
date: 2026-09-17T07:24:40+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "AI Agent"]
summary: "Anyone who writes Kotlin has been carrying the same nagging feeling for the past two years."
cover:
    image: "/images/adk-kotlin-10/cover.png"
    alt: "Kotlin Developers Can Finally Write Production-Grade AI Agents: A Deep Dive into Google ADK for Kotlin 1.0"
    relative: false
---

## What an agent framework with a KMP core actually means

Anyone who writes Kotlin has been carrying the same nagging feeling for the past two years.

The AI agent wave keeps rolling in, but open the source and look: LangChain, CrewAI, AutoGen, all Python. If you want to build an agent inside the Kotlin ecosystem you already know, you either hand-roll it on an HTTP client, or write a Python service and call it from Kotlin, with a process boundary and serialization sitting in between for good measure.

On September 9, Google filled that gap: **Agent Development Kit (ADK) for Kotlin 1.0 is officially GA**.

And this is not a straight port of the Python version. Underneath it is a **Kotlin Multiplatform (KMP) core**.

That detail matters. More on it later.

## First, what this thing actually is

ADK for Kotlin's positioning is clear: **fully decoupled from the model backend, session storage, and memory system**.

In plain terms: you write the agent logic once in Kotlin, and the model can be Gemini or something else entirely; sessions can live in memory, in Room, or in the cloud, and you swap them as you like.

1.0 is feature-aligned with ADK Python and Java, and it brings over the multi-agent patterns that matured in the Python ecosystem:

- **Hierarchical multi-agent**: agents chain together and delegate tasks to specialized sub-agents
- **Context compaction**: history that grows too long gets summarized automatically, keeping tokens inside the window
- **Human-in-the-loop**: sensitive operations pause and wait for user confirmation before continuing
- **Resumable sessions**: agent interaction state serializes and restores across sessions
- **Java interop**: existing Java applications can call Kotlin-written agents directly
- **Vertex AI integration**: enterprise Session, RAG Memory, and Memory Bank services

One of these deserves its own paragraph: **context compaction**.

Anyone who has written an AI chat client knows that managing history across turns is unavoidable work. A sliding window loses information, and stuffing everything in blows the token budget. ADK makes it a built-in framework capability, so you do not have to build that wheel yourself.

## The technical part: why this counts as Kotlin-native

This is the part of ADK for Kotlin I find most interesting.

### Tools as annotations, schemas generated at compile time

Python agent frameworks generally lean on runtime reflection to read function signatures and generate tool descriptions. ADK for Kotlin takes another road: **generation at compile time with KSP (Kotlin Symbol Processing)**.

The code you write is just an ordinary Kotlin function:

```kotlin
class InfrastructureDiagnosticsService {

    @Tool
    suspend fun getServiceMetrics(
        @Param("Target service or database cluster") serviceName: String,
        @Param("Time window in minutes") windowMinutes: Int? = 15,
    ): ServiceMetrics {
        // query the monitoring backend: Datadog, Prometheus, Cloud Monitoring
        return ServiceMetrics(
            serviceName = serviceName,
            cpuUsagePercent = 91.4,
            connectionPoolUsagePercent = 98.5,
            p99LatencyMs = 2450,
            errorRatePercent = 4.2,
        )
    }
}
```

KSP generates a `generatedTools()` extension function at compile time.

**Three things follow from that**: type safety (the schema cannot drift away from the signature), suspend function support (coroutines fit in naturally), and zero runtime reflection (fast startup, small binary, which matters a lot on Android).

If you are used to compile-time checking in Kotlin, this design direction should resonate: **anything the compiler can catch, do not defer to runtime**.

### Skills: progressive disclosure

ADK has a concept called a Skill, and I think it is cleverly designed.

You write a standard operating procedure as `SKILL.md` and drop it into resources:

```markdown
---
name: database-incident-triage
description: Standard operating procedure for diagnosing database latency spikes.
allowed-tools: [getServiceMetrics, fetchRecentDeployments, notifyOnCall]
---

# Database Incident Triage SOP

1. **Telemetry**: Call `getServiceMetrics` to inspect CPU, latency, and pool saturation.
2. **Correlation**: Call `fetchRecentDeployments` to check for recent changes.
3. **Safety Rules**: Never restart primary nodes during peak hours.
4. **Notify**: Broadcast diagnosis to `#production-alerts`.
```

The agent only loads that SOP into context when it needs it, and bundled resource files such as `assets/mitigation_rules.txt` are pulled on demand too.

The team calls this **progressive disclosure**: **spend tokens where they matter, do not dump every piece of knowledge into the model up front**.

## Hands on: a database incident triage agent

The official example is a practical one. Suppose production fires a database alert and we want an agent to help triage it:

```kotlin
object IncidentTriageDemoAgent {
    val rootAgent = LlmAgent(
        name = "incident_triage_agent",
        model = Gemini(name = "gemini-3.8-flash"),
        instruction = Instruction(
            """
            You are an SRE on-call diagnostic assistant.
            1. Load the matching triage SOP using `load_skill`.
            2. Follow the playbook steps strictly.
            3. Use your diagnostics tools to inspect telemetry and notify the team.
            """.trimIndent()
        ),
        // compile-time generated function tools (zero reflection)
        tools = InfrastructureDiagnosticsService().generatedTools(),
        // dynamic skill toolset
        toolsets = listOf(SkillToolset(NewFileSystemSource(resolveSkillsDir()))),
    )
}
```

Run it:

```kotlin
fun main() = runBlocking {
    val runner = InMemoryRunner(
        agent = IncidentTriageDemoAgent.rootAgent,
        appName = "IncidentTriageApp"
    )

    val alert = "ALERT [P1]: Database latency spike detected on 'users-postgres-cluster'!"

    val events = runner.runAsync(
        userId = "oncall-sre",
        sessionId = UUID.randomUUID().toString(),
        newMessage = Content.fromText(Role.USER, alert)
    ).toList()

    for (event in events) {
        event.content?.parts?.firstOrNull()?.text?.let { println("Agent: $it") }
    }
}
```

Note the `runBlocking` plus streaming events. **Coroutines are a first-class citizen here**, not something bolted on afterwards.

On receiving the alert, the agent walks the whole flow by itself: load the SOP, call `getServiceMetrics()` and find the connection pool at 98.5%, call `fetchRecentDeployments()` and pin it to a deploy 25 minutes ago, post the diagnosis to the alert channel, recommend a rollback.

That is a complete loop of **tool calls, domain knowledge, and autonomous decision-making**.

## On-device Android: it is not cloud-only

If you work on mobile, the Android extensions may be the more exciting part for you.

The hard part of mobile AI is the balance: cloud inference is capable, but latency, privacy, and offline availability are all problems. ADK for Kotlin 1.0 ships a modular Android story:

- **LiteRT-LM**: small models on device
- **ML Kit (beta)**: local inference
- **Firebase AI Logic**: hybrid cloud and on-device orchestration
- **Room persistence**: history in SQLite, survives process death
- **AppSearch indexed memory**: an on-device full-text memory service
- **FileArtifactService**: generated files such as statements and receipts land directly in local storage

The official financial assistant sample wires all of it together:

```kotlin
fun createAndroidRunner(applicationContext: Context, agent: LlmAgent): InMemoryRunner {
    return InMemoryRunner(
        agent = agent,
        appName = "AndroidFinancialApp",
        // SQLite persistence: sessions survive process death
        sessionService = RoomSessionService.fromContext(applicationContext),
        // on-device full-text indexed memory
        memoryService = AppSearchMemoryService.fromContext(applicationContext),
        // app-private file storage
        artifactService = FileArtifactService.fromExternalFilesDir(applicationContext)
    )
}
```

And HITL is handled pragmatically. For something sensitive like a transfer, you just add a flag when defining the tool:

```kotlin
@Tool(
    name = "transferFunds",
    description = "Transfers money to another account. Requires explicit user approval.",
    requireConfirmation = true
)
fun transferFunds(
    @Param("Recipient account ID") recipientId: String,
    @Param("Amount in USD") amount: Double
): String { /* ... */ }
```

When execution reaches it, the agent **pauses** and waits for the user to tap confirm in the UI.

**If you build finance or healthcare apps, this is close to non-negotiable.**

## What it means for KMP developers

Back to that KMP core from the top of the post.

ADK for Kotlin's core engine is cross-platform and decoupled from the model backend, session storage, and memory. The implication of that architectural choice: **agent business logic is shareable**.

Picture it. Your agent orchestration, tool definitions, and SOP skill files run on Android, and they also run on a server JVM. If iOS support lands later, that is one codebase across three targets — which is exactly the story KMP has been telling all along.

Combine that with what we covered on **Kotlin 2.4.20** (sealed class mapping in Swift export, cross-language inheritance), and the puzzle pieces on Kotlin's cross-platform and AI tracks have visibly started fitting together this year.

For anyone building an AI chat client there is a more direct takeaway: **the framework-level pieces — session persistence, context compaction, HITL — already have mature implementations**. You do not have to build them from scratch.

## How to start

Just add the dependencies:

```kotlin
dependencies {
    // core engine + KSP processor
    implementation("com.google.adk:google-adk-kotlin-core:1.0.0")
    ksp("com.google.adk:google-adk-kotlin-processor:1.0.0")

    // Android on-device extensions (optional)
    implementation("com.google.adk:google-adk-kotlin-mlkit-android:1.0.0-beta")
    implementation("com.google.adk:google-adk-kotlin-litertlm:1.0.0")
    implementation("com.google.adk:google-adk-kotlin-firebase-android:1.0.0")
}
```

Official resources:

- 📖 Docs and getting started: adk.dev
- 🐙 GitHub: github.com/google/adk-kotlin
- 📱 Android guide: Building Android agents with ADK
- ☕ Java interop sample: ADK Kotlin Java sample

## Wrapping up

I do not think the point of ADK for Kotlin 1.0 is that there is one more agent framework on the pile. The point is this:

**The Kotlin ecosystem finally has an agent development story that is compile-time safe, coroutine-native, and reusable across platforms.**

It does not drag Python's patterns over. It redesigned around Kotlin's language features: KSP instead of reflection, coroutines instead of callbacks, a KMP core instead of a single-platform implementation.

If you work in KMP, this is worth a serious look.

---

*Reference: [Announcing ADK for Kotlin 1.0 - Google Developers Blog](https://developers.googleblog.com/en/announcing-adk-for-kotlin-10-building-production-ready-ai-agents-in-kotlin-android-and-beyond/) (2026-09-09, by Guillaume Laforge)*
