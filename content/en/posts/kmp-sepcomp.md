---
title: "Kotlin Multiplatform Compiling \"Whenever It Feels Like It\" -- Officially Acknowledged"
date: 2026-10-03T22:24:18+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "If you write Kotlin Multiplatform you have probably hit this: the line is red in the IDE, underlined with a cannot find symbol, you hesitate for ages and do not dare commit, then you build and it is green, and it even runs correctly."
cover:
    image: "/images/kmp-sepcomp/cover.png"
    alt: "Kotlin Multiplatform Compiling \"Whenever It Feels Like It\" -- Officially Acknowledged"
    relative: false
---

## Start with that moment

If you write Kotlin Multiplatform you have probably hit this: the line is red in the IDE, underlined with a cannot find symbol, you hesitate for ages and do not dare commit, then you build and it is green, and it even runs correctly.

We used to blame the IDE acting up, an index that had not been built, just clear the caches. On September 28 JetBrains published a post, and the first line is blunt:

> The current compilation approach to Kotlin Multiplatform projects works, but sometimes can lead to unexpected or hard-to-predict behavior.

In plain terms: the current compilation approach works, but sometimes it hands you behavior you cannot predict. Then it lists two examples, both things we have hit and never explained:

> The IDE analysis disagrees with the compiler on an overload, type inference, or whether the code should compile at all – with the IDE being stricter than the compiler.

> Your `commonTest` code can unexpectedly call something from a platform source set, breaking your assumptions.

One is the IDE being stricter than the compiler, the other is commonTest quietly reaching into a platform source set. Below I explain the mechanism first, then give my own test results, because in the official example you only get shown half the price of the fix.

## Why the disagreement exists

The key is what common code resolves its dependencies against when it compiles.

In a KMP module, commonMain is not compiled into an independent artifact and that is the end of it. When compiling the JVM target, the compiler takes commonMain and jvmMain together and compiles them against the dependency's platform artifact (a jar on JVM). So a call in commonMain has a chance to implicitly land on a declaration in the dependency's jvmMain.

The IDE's code analysis does not do this. It assumes common code can only see the dependency's common declarations, the part listed in the metadata KLIB. jvmMain declarations are not in there, so the IDE shows red.

Two different understandings of the same line, and neither is wrong, but you are stuck in the middle. The post even gives a nastier version: inconsistent overload selection, where the IDE jumps to the commonMain overload while the runtime prints the platform one.

Then there is type inference divergence, where the common side infers Any and JVM compilation fails outright with Unresolved reference. The reverse happens too: with the same example and the flag on, the common phase fixes the return type of currentEvent() to Event, `.name` has a basis, and JVM compilation passes. So this feature cuts both ways. It makes code that got lucky fail, and it also fixes code that used to fail.

The difference between those two paths looks like this:

![](/images/kmp-sepcomp/lookup-paths.png)

## The new scheme is one flag

Kotlin 2.5.0-Beta1 introduced an optional separate compilation mode, so the compiler also uses the strict metadata KLIB resolution when compiling common source sets, lining up with the IDE.

```properties
# gradle.properties
kotlin.kmp.separateCompilation=true
```

It is marked Experimental and off by default. One bonus: it makes incremental compilation of common source sets possible, which was not achievable before.

The official example reads easily. Code the IDE flagged red but the compiler accepted now fails to compile too; just wire up expect/actual explicitly and you are fine.

## Test one: it really does stop you

I built a minimal KMP project locally (Kotlin 2.5.0-Beta1, two targets: jvm and js), two modules:

```text
lib/  src/commonMain/Marker.kt      # interface only
      src/jvmMain/Foo.kt            # class Foo, declared only in the platform source set
app/  src/commonMain/Main.kt        # directly new a Foo from lib
```

app depends on lib, and app's commonMain news up a class that only exists in lib/jvmMain.

Flag off:

```text
> Task :app:compileKotlinJvm
BUILD SUCCESSFUL in 30s
```

Flag on, same code:

```text
e: app/src/commonMain/kotlin/Main.kt:3:12 Unresolved reference 'Foo'.
e: app/src/commonMain/kotlin/Main.kt:6:13 Unresolved reference 'Foo'.
BUILD FAILED in 2s
```

The post was right, and it stops you cold. Not a warning.

Same for commonTest. I put a PlatformOnly class in app's jvmMain and had commonTest new it: flag off BUILD SUCCESSFUL, flag on gives `Unresolved reference 'PlatformOnly'`.

## Test two: the price of the fix the official example does not show

The official fix is explicit expect/actual. lib's commonMain declares expect, the platform source set provides actual. I made that change and the JVM side really did pass:

```kotlin
// lib/commonMain
expect class Foo()

// lib/jvmMain
actual class Foo actual constructor()
```

With the flag on, `:app:compileKotlinJvm` is BUILD SUCCESSFUL in 1s. That half of the official story is right.

Then I went to compile the JS side of the same module:

```text
e: lib/commonMain/kotlin/Marker.kt:5:1 The 'expect' declaration 'lib.Foo'
   has no 'actual' declaration in module '<commonMain> for JS'.
BUILD FAILED in 2s
```

expect/actual is a paired contract. Every target needs its own actual. The blog example only ever showed the JVM side. Turn it on in a multiplatform project and you owe one actual per target.

## One thing the official post does not mention: the version number is not the gate

The blog says the feature was introduced in Kotlin 2.5.0-Beta1. I took the same bad code and tried it on Kotlin 2.4.20 (current stable):

```text
# without the property
BUILD SUCCESSFUL in 704ms

# with kotlin.kmp.separateCompilation=true
e: app/src/commonMain/kotlin/Main.kt:3:12 Unresolved reference 'Foo'.
BUILD FAILED in 11s
```

The property still flips behavior on 2.4.20, and I checked that this project really resolves to kotlin-gradle-plugin:2.4.20.

So this flag is not something that came into existence in 2.5, it is just that the team only started talking about it as a feature in 2.5.0-Beta1. Flip it around: do not treat the version number as a gate, and do not treat it worked on 2.4 as a conclusion either. Neither is solid. You have to run it on your own project to know.

## A quick reference table

Here is everything above plus the official Known issues section in one table, for when you hit one of these cases:

| State of the old code | Result under the new scheme | What to do |
|---|---|---|
| commonMain sneaks in a declaration from a dependency's jvmMain / nativeMain | compile fails with Unresolved reference | have the library expose that capability as expect/actual, or push the call down into the platform source set |
| commonTest sneaks in platform code like jvmMain | test compilation fails | expose a public interface via expect/actual, or move the test to jvmTest |
| same-named overloads in common and on the platform side | the chosen overload may change (my test: from platform to common) | check which overload these calls actually resolved to before |
| common type inference affected by extra supertypes on the platform actual | the inferred result gets pinned down, which may fix an old error or expose code relying on the old behavior | declare type boundaries explicitly on the common side |

## Should your project turn it on

Combining the above with the official Known issues section, my checklist is this:

- Do the libraries you depend on properly publish metadata KLIB? The new scheme turns that from nice to have into required. A library whose author cut corners will fail the moment you turn it on, and that is not your fault.
- How much does the project rely on common calling platform? The KMP project I have routes platform differences through interfaces and barely uses expect/actual, so before turning it on I need to sweep for these calls, otherwise a pile of Unresolved reference appears all at once.
- Do you use cinterop? The docs call out two known issues here (KT-88178, KT-41509), so projects with native C interop should be careful.
- Modules with only one target are unaffected, the docs say so plainly.
- It is still Experimental and off by default. Treat it as a preview, not an upgrade.

## A word on the iOS front

In the same week JetBrains also published a TeamCity design retrospective about turning Kotlin Multiplatform iOS packaging into a guided flow: install a plugin, push, and the IDE pops a Configure CI prompt. Follow it and you go from first push to running tests, signing, shipping to TestFlight, and finally hooking up GitHub to run on every push.

Along the way it generates four files (`teamcity.yaml` plus Fastfile, Appfile, Gemfile) and leaves them local and uncommitted. The first build runs on a hosted macOS agent, and only after you have seen it pass does it let you commit.

I did not test this part, there is no TeamCity account on this machine. That section comes entirely from the official design retrospective and the official tutorial it points to.

## Last

Back to that moment where the IDE shows red and the compiler passes. It was never an IDE bug and never your imagination. Two sets of resolution rules simply were not aligned for a long time. What the team is doing now is pulling the compiler over to the IDE's side: stricter, more predictable, at the price of code that used to get lucky having to be made explicit first.

What I want to do next is turn it on in the KMP project I have, list every Unresolved reference that pops up, and see how much expect/actual that project actually owes.

Have you turned this flag on in your Kotlin Multiplatform project? How many errors did it report? Tell me in the comments.

References:
- A More Reliable Compilation Scheme for Kotlin Multiplatform Modules: https://blog.jetbrains.com/kotlin/2026/09/a-more-reliable-compilation-scheme-for-kotlin-multiplatform-modules/
- Designing the Kotlin Multiplatform and TeamCity Integration: https://blog.jetbrains.com/teamcity/2026/09/designing-the-kotlin-multiplatform-and-teamcity-integration/
- Official iOS CI/CD tutorial: https://kotlinlang.org/docs/multiplatform/ios-ci-cd-teamcity.html
