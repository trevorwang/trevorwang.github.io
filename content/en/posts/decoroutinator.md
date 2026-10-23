---
title: "Your Coroutine Crashed and the Stack Is Blank: Getting the Lost Call Chain Back with Decoroutinator"
date: 2026-09-23T07:37:31+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "Production threw an IllegalStateException. You open the stack trace and see this:"
cover:
    image: "/images/decoroutinator/cover.png"
    alt: "Your Coroutine Crashed and the Stack Is Blank: Getting the Lost Call Chain Back with Decoroutinator"
    relative: false
---

Production threw an `IllegalStateException`. You open the stack trace and see this:

```
java.lang.IllegalStateException: user 42 not found
	at demo.ChainKt.fetchUser(Chain.kt:11)
	at demo.ChainKt$fetchUser$1.invokeSuspend(Chain.kt)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)
	at kotlinx.coroutines.DispatchedTaskKt.resume(DispatchedTask.kt:233)
	at kotlinx.coroutines.DispatchedTaskKt.dispatch(DispatchedTask.kt:152)
	... the next dozen lines are all coroutine internals
```

One line of your own code, then nothing but framework. You know where it blew up, but not **who** called it. `fetchUser` is called from several places in this project, and any of them could have passed in 42.

This is not bad luck. It is how coroutines normally behave. And fixing it takes one dependency.

## Why the call chain disappears

The whole point of coroutines is writing async code that looks synchronous. The compiler turns every suspend function into a state machine. When the function "returns," it is not really returning on the JVM call stack; it hands the result to a continuation and lets the dispatcher decide where and when to continue.

The problem is in that "continue." At the moment of suspension, the frames in between have already been popped off the JVM stack. Resumption starts on a brand new stack: it begins at the dispatcher and jumps straight into `fetchUser`'s continuation. `loadProfile` and `renderScreen` leave no frames on that stack at any point, so the exception has no way to find them.

So losing frames is not a bug. It is the inevitable result of the mechanism. The Kotlin team knows this and has offered a fix: turning on `kotlinx.coroutines.debug` enables stacktrace recovery.

It just does not save you here. I ran this example locally and confirmed the property made it into the app JVM:

```
Picked up JAVA_TOOL_OPTIONS: -Dkotlinx.coroutines.debug=on
java.lang.IllegalStateException: user 42 not found
	at demo.ChainKt.fetchUser(Chain.kt:11)
	at demo.ChainKt$fetchUser$1.invokeSuspend(Chain.kt)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)
```

`loadProfile` and `renderScreen` are still missing. The official mechanism only covers some scenarios, and suspend-then-resume — the most common path of all — happens to be outside them.

## Decoroutinator takes a different angle

[Stacktrace-decoroutinator](https://github.com/reformator14/stacktrace-decoroutinator) goes at it by **replacing the coroutine resumption implementation wholesale**. The author puts it plainly:

> Decoroutinator replaces the coroutine resumption implementation. It generates methods at runtime with names that match the entire coroutine call stack.

> These methods do nothing except call each other sequentially in coroutine call stack order, creating real JVM stack frames.

In plain terms: it does not try to "guess" the call chain out of the continuation. It **actually builds those frames**. If exceptions need the JVM stack to find the caller, then give the stack something to find.

The project is Apache-2.0, 374 stars, has taken Kotlin Foundation grants, shipped 2.6.6 on September 10, and the repo had commits in mid-September. For a small tool, that is a healthy maintenance state.

## The result first, installation second

I dropped the same source file into three modules, the only difference being whether Decoroutinator is on. First run, the original version:

```
$ ./gradlew :plain:run
java.lang.IllegalStateException: user 42 not found
	at demo.ChainKt.fetchUser(Chain.kt:11)
	at demo.ChainKt$fetchUser$1.invokeSuspend(Chain.kt)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)
	at kotlinx.coroutines.DispatchedTaskKt.resume(DispatchedTask.kt:233)
	...
```

21 lines of stack, and only 4 frames belong to the `demo.` package, two of which are the same function.

With the decorator on:

```
$ ./gradlew :decoroutinated:run
java.lang.IllegalStateException: user 42 not found
	at demo.ChainKt.fetchUser(Chain.kt:11)
	at demo.ChainKt$fetchUser$1.invokeSuspend(Chain.kt)
	at kotlin.coroutines.jvm.internal.DecoroutinatorBaseContinuationAccessorImpl.invokeSuspend(base-continuation-accessor.kt:15)
	at dev.reformator.stacktracedecoroutinator.common.internal.Provider.callInvokeSuspendIfResultIsNotCoroutineSuspended(provider-impl.kt:123)
	at dev.reformator.stacktracedecoroutinator.provider.SharedSpec.$decoroutinator$resume(provider-api.kt:106)
	at demo.ChainKt.loadProfile(Chain.kt:15)
	at demo.ChainKt.renderScreen(Chain.kt:21)
	at demo.MainKt$main$1.invokeSuspend(Main.kt:8)
	...
```

31 lines, 7 frames in `demo.`. What matters is not that it got longer. It is that these two lines came back:

```
	at demo.ChainKt.loadProfile(Chain.kt:15)
	at demo.ChainKt.renderScreen(Chain.kt:21)
```

`renderScreen` line 21 called `loadProfile`, and `loadProfile` line 15 called `fetchUser`. A complete call chain, with line numbers. Now you can see at a glance who passed in 42.

Not a character of the code changed. Here are the three ways to install it.

## On the JVM, one plugin line is enough

```kotlin
plugins {
  kotlin("jvm") version "2.3.21"
  application
  id("dev.reformator.stacktracedecoroutinator") version "2.6.6"
}
```

This is the least work. The plugin generates helper methods into your classes at build time, and you need no extra JVM flags at runtime.

The same library has two other ways to enable it, both going through JVM instrumentation instead of build-time generation:

```bash
# Option two: JVM startup flag
-javaagent:/path/to/stacktrace-decoroutinator-jvm-agent-2.6.6.jar
```

```kotlin
// Option three: add the dependency, then call this once before any coroutine starts
import dev.reformator.stacktracedecoroutinator.jvm.DecoroutinatorJvmApi

fun main() {
  DecoroutinatorJvmApi.install()
  // run your code as usual after this
}
```

The coordinates are `dev.reformator.stacktracedecoroutinator:stacktrace-decoroutinator-jvm:2.6.6`.

Option three has a side effect worth mentioning up front: it works by **dynamically loading** a Java agent, and the JVM prints a warning at you —

```
WARNING: A Java agent has been loaded dynamically
WARNING: Dynamic loading of agents will be disallowed by default in a future release
```

Today it is only a warning. Future releases will disallow it by default. If your environment cares about that, options one and two are safer.

## In a KMP project, the plugin does nothing

This is the pitfall I hit hardest this round, and the main reason I wanted to write this for KMP readers.

Seeing the plugin, the first instinct is to apply it to the multiplatform module. I did that, and the stack **did not recover at all**. Same as no install. Worse, it threw a different error along the way:

```
Exception in thread "main" java.lang.NoClassDefFoundError:
dev/reformator/stacktracedecoroutinator/provider/BaseContinuationExtractor
	at demo.MainKt.main(Main.kt:6)
```

Which means the plugin did rewrite bytecode, but left the classes its own runtime needs outside the classpath. I manually added the three artifacts the plugin injects into a normal `kotlin("jvm")` module, and the `NoClassDefFoundError` went away — **but the stack still did not recover.**

The reason is not hard to work out: the plugin only wires itself into a plain JVM module's compile task, and a KMP jvm target's compile task is outside what it takes over. So the bytecode moved a little, and the part that actually generates the "fake call chain" never ran.

For a KMP project the right move is to skip build-time generation and use runtime instrumentation instead. Both of these are tested and working:

```kotlin
kotlin {
  jvmToolchain(21)
  jvm()
  sourceSets {
    jvmMain.dependencies {
      implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.11.0")
      // Option three needs this; drop the line if you are using the agent
      implementation("dev.reformator.stacktracedecoroutinator:stacktrace-decoroutinator-jvm:2.6.6")
    }
  }
}

// Option two: the agent goes only into the run args, no source or dependency changes
val decoroutinatorAgent by configurations.creating
dependencies {
  decoroutinatorAgent("dev.reformator.stacktracedecoroutinator:stacktrace-decoroutinator-jvm-agent:2.6.6")
}
tasks.register<JavaExec>("runKmp") {
  mainClass.set("demo.MainKt")
  classpath = files(kotlin.targets.getByName("jvm").compilations.getByName("main").output.allOutputs,
                    configurations.getByName("jvmRuntimeClasspath"))
  jvmArgs("-javaagent:${decoroutinatorAgent.singleFile.absolutePath}")
}
```

Both produced recovered stacks inside the KMP module: `loadProfile` and `renderScreen` present, 7 frames in `demo.`.

One more thing has to be said clearly: **Native and JS targets are out, and it is not a configuration problem**. The library works by building stack frames on the JVM. No JVM, no mechanism. The author was asked in issue #5 whether a multiplatform version was planned, and answered that there is no plan for now. That issue has been open since 2022.

So the real choice for a KMP project is: install it for the JVM/Android side, leave iOS alone. If you are mostly chasing iOS crashes, this article will not help you.

## Known pitfalls

The README lists a few, and every one of them is real:

- Android only supports the plugin route, and it depends on `MethodHandle`, so it needs API 26 (Android 8) or above. On older versions it **silently turns recovery off** instead of crashing, so it is safe to include.
- R8/ProGuard breaks it. Obfuscation renames the classes the runtime needs, so add `decoroutinatorAndroidProGuardRules()` to your release rules.
- Coverage disappears when used with Jacoco. This is a Jacoco limitation; the fix is to make Jacoco's agent run before Decoroutinator's.
- The Shadow plugin has a known conflict ([shadow#882](https://github.com/GradleUp/shadow/issues/882)). If you build a fat jar, read the workaround first.
- Robolectric puts its classes in a separate classloader, which makes tests fail. You need to exclude `dev.reformator.stacktracedecoroutinator.` in `packagesToNotAcquire`.

Also, if you only want it in tests, the official advice is to apply the plugin in a separate subproject for tests. If that is hard to split out, there is a switch:

```kotlin
stacktraceDecoroutinator {
  androidTestsOnly = true
}
```

To check whether it actually got installed at runtime, query the status:

```kotlin
DecoroutinatorCommonApi.getStatus { it() }
```

On Android it can also bundle `DebugProbes` for you, via `embedDebugProbesForAndroid = true`, which lets you dump coroutine state at runtime.

## Should you install it

I lean yes, especially if someone on the team is still locating coroutine bugs by adding logs and guessing.

What it does is narrow: put real frames on the JVM stack. No API changes, no invasive refactor, and removing it is a one-dependency change. The cost is one more instrumentation layer at runtime, plus the pitfall list above — most of which only show up in Android packaging. A pure JVM project barely notices.

One caution: it changes the **resumption implementation** itself. So run a pass in staging before shipping to confirm there is no behavior difference, rather than flipping it on in your critical path.

---

The full comparison project is at https://github.com/trewvip-arch/decoroutinator-kmp-demo

All three modules share one source file, `plain` / `decoroutinated` / `kmp`. Run it once and you see the original of every stack output in this article:

```bash
./gradlew :plain:run          # not installed, chain is broken
./gradlew :decoroutinated:run # plugin route, chain intact
./gradlew :kmp:runKmp         # KMP + agent, chain intact
./gradlew :kmp:runKmpInstall  # KMP + install(), chain intact
```

Environment: Stacktrace-decoroutinator 2.6.6, Kotlin 2.3.21, kotlinx-coroutines 1.11.0, JDK 21, macOS. Every stack output here was actually run.
