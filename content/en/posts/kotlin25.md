---
title: "Kotlin 2.5 Changes Its Publishing Format, and the Compiler Gets Stricter"
date: 2026-10-22T07:33:31+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "On October 21, Kotlin 2.5.0-Beta2 shipped. Most people open the notes, read the four promoted and two experimental language features, decide it is unremarkable, and close the tab."
cover:
    image: "/images/kotlin25/cover.png"
    alt: "Kotlin 2.5 Changes Its Publishing Format, and the Compiler Gets Stricter"
    relative: false
---

On October 21, Kotlin 2.5.0-Beta2 shipped. Most people open the release notes, read the language feature lines: four promoted, two experimental. Then they decide this version is unremarkable and close the tab.

The real changes are in the later sections. The publishing format changed, the compilation scheme changed, and three defaults were quietly tightened. I installed both compilers on the same machine and fed each one the same source. The results differ.

## The point of this release is not syntax

Two of the four stable features are really about turning a choice into a default: the syntax-only mode for destructuring by name, and the unused return value checker. Not one line of old code needs changing. What new code can express changes.

The two experimental ones follow the same line: companion extensions and companion blocks, and full value classes. All of them remove a bit of arbitrary restriction from the language.

The language layer is the quiet part this time. Laid out properly, the substance is in the last three sections of the doc: the Kotlin Multiplatform publishing format, the new compilation scheme, and incremental compilation on Apple platforms. None of those change what you write. They change what you ship and how long it takes to build.

## One .kar file replacing seven thousand

Their words:

> The new format publishes platform artifacts (except for JVM and Android) as a single .kar archive, which is a ZIP archive additionally compressed with XZ.

A library supporting several targets used to publish several sets of artifacts. Now everything except JVM and Android goes into a single `.kar`, which is a ZIP with an extra layer of XZ on top.

The numbers are clearer than the description. Their example is cryptography-kotlin: published artifacts went from 7,539 files and 91 MB to 1,036 files and 23 MB. For the whole multi-module project.

![The release notes text, with that set of numbers boxed in red](/images/kotlin25/fig1_kar.png)

One line to turn it on. Per module in the build script, project-wide in `gradle.properties`:

```
kotlin.publicationFormat=KOTLIN_ARCHIVE
```

Two caveats have to be read together. Except JVM and Android: those two ecosystems still use standard Maven artifacts. The consumer side is fully available, so when a library you depend on switches format, your build does not move. The publishing side is still experimental, so if you publish libraries yourself, verify your pipeline separately.

## Why Maven Central's limit is about file count

The notes only say Maven Central recently introduced a publishing size limit. What sits behind that sentence is more interesting than size.

What blocks library authors is not total bytes, it is file count. A library with six targets, one artifact set per target, plus source jars and metadata, lands in the thousands without trying. Every file is a separate record in the repository, a separate upload, a separate index entry. For the repository that is operational cost. For the user it is the first screen of requests during dependency resolution.

So 7,539 files down to 1,036, minus eighty-six percent, matters more than 91 MB going to 23 MB. What you save is requests and index entries, not bandwidth.

JVM and Android being excluded says something too: those two ecosystems are still anchored to standard Maven classifiers and cannot move. The real situation for cross-platform libraries is many targets, mainline platforms on the old format, everything else in an archive. That hybrid will stick around for a while.

The change is completely transparent downstream, and because it is transparent, it can be recommended to everyone. The beneficiaries are library authors and the people maintaining release pipelines, not those of us writing application code.

## The new compilation scheme: common code should not quietly reach platform APIs

This section is written with restraint. Four items: make the compiler treat common source sets the same way the IDE does; prevent common code from accidentally resolving to platform-specific declarations; improve consistency of overload resolution and type inference; enable incremental compilation for common source sets.

The first three fix an old hole. One wrong import in `commonMain` and the IDE marks it red while the command line passes, or the reverse, and the two never agree. The cause: the IDE and the compiler each have their own resolver, each doing its own math.

Unifying them has a side effect: code that used to compile may not anymore. That is not a regression. That code was reaching symbols it should not have seen. Nobody was stopping it.

The fourth is the everyday payoff. Common code finally gets incremental compilation, and people who live in the IDE feel it most. On CI you will not notice, because the pipeline builds from scratch every time.

JetBrains wrote a dedicated post on this scheme in September, with the word reliable in the title. The problem being fixed is exactly unreliability: the IDE and the compiler reaching two different conclusions about the same common code.

## The compiler got stricter: three defaults changed

The most visible one first. The same code fails to compile on 2.4.21 and passes by default on 2.5.0-Beta2:

```
(val mail = email, val name = username) = user
```

`val` inside the parens, an equals sign picking which property to take, and the variable name no longer has to match the property name.

![All three features on my machine](/images/kotlin25/fig3.png)

An old hole while we are here. Positional destructuring like `val (email, username) = user` passes on both versions, but the values are assigned by position. I destructured `User(username, email)` into `email=alice`, `username=alice@example.com`. Every variable name wrong, and the compiler said nothing. This feature exists because of that.

Second, the unused return value checker. The same file passes silently on 2.4.21 and warns on 2.5.0-Beta2: `unused return value of 'plus'`. If an expression produces a non-Unit value and nobody uses it, it tells you.

Third, cross-module inlining. Optional when introduced in 2.4.0, on by default in 2.5.0-Beta2. The off switch is `-Xklib-ir-inliner=disabled`.

There is one more that is not a default. Type inference generates fewer constraints and no longer stalls on complex generics. The notes are restrained, saying only that it may change inference results, overload selection and diagnostics. In plain terms: after upgrading, some code will infer different types, and that code is code you previously believed was correct. Off switch: `-XXLanguage:-EliminateSecondKindIncorporation`.

## value class is where the cut goes deepest

Value classes used to allow exactly one property and required `@JvmInline`. Multiple properties work now, plus two new shapes: abstract value classes and value objects.

On my machine: 2.4.21 rejects a multi-property value class outright, telling you to enable `-XXLanguage:+FullValueClasses`. 2.5.0-Beta2 passes with `-Xfull-value-classes`.

Same for value objects. 2.4.21 says `modifier 'value' is not applicable to 'standalone object'`; 2.5 compiles and runs with the flag.

One detail explains it better than the docs. Write `===` on a value object and the compiler errors immediately. Identity comparison is banned by the language, because it has no identity to compare.

The real trap is migration. `value class` is no longer inlined by default. The stated timeline: warnings start in 2.5.20, errors in 2.6 or 2.7. On 2.5.0-Beta2 the switch is not even recognized yet; the compiler answers that the flag is not supported, which means the window really does open at 2.5.20.

Meaning: upgrading today will not blow up. But you have until 2.5.20 to mark the old code, deciding what keeps inlining and what lets go. Whether a value class wrapping a single String should inline is a trade-off to begin with, and the language should not make that default for everyone.

## The standard library closes two small gaps

Three functions landed for `if` expressions: `onTrue()`, `onFalse()`, `ifOrNull()`. Printing a line when an add succeeds used to mean a local variable and an if/else. Now one line holds it:

```
val added = tags.add("wasm").onTrue { println("Tag added") }
```

The return value is still the same boolean. The check and the side effect sit on one chain, and you no longer build a variable for a case this small.

The other half is Unicode code points. A `Char` holds one UTF-16 code unit, so when you hit a surrogate pair like an emoji, iterating by `Char` splits one character in half. The new `CodePoint` is a value class holding the code point in an Int, and it can tell you how many UTF-16 units it occupies, which category it belongs to, whether it is a letter.

Both need `@OptIn(ExperimentalStdlibApi::class)`. Note the shape of `CodePoint`: it is only writable because value classes were relaxed. Language and standard library line up in this release.

## Two surprises on my machine

![The same code through both compilers](/images/kotlin25/fig2.png)

First surprise: compilation did not get faster. On a 489-line project, warmed up, 2.4.21 has a median of 2.86 seconds and 2.5.0-Beta2 comes in at 2.99. The output is 161 KB either way. The difference sits in the noise, so it is not slower, but there is no visible payoff from the inference work on a small project. That kind of optimization only shows up in large generic-heavy code.

Second: the scope of that switch. `-Xklib-ir-inliner=disabled` is rejected outright by the JVM compiler, which says the flag is not supported. It only means something where klib is produced: Native, JS, Wasm.

On by default in the docs has to be read per platform.

## Three things to do before upgrading

The JDK floor moved. The compiler's minimum went from 8 to 17, and anything below now errors out. CI images and Docker base images have to follow, and that step usually eats more time than the code changes.

If you publish Kotlin Multiplatform libraries, do not rush into `.kar`. The publishing side is still experimental. Try it in a pipeline, do not push it to a real release.

Run your tests. The unused return value check and the inference changes both fall into the passes silently but behaves differently category. Reading the docs will not reveal the problem; running tests will.

## Last

Most of what 2.5 does lands on defaults: packaged by default, checked by default, inlined by default, destructured by name by default. Every one of those comes with an off switch, which says the team knows it will step on people, and also that these defaults are not final.

Do you have code the IDE marks red while the command line compiles it fine? After 2.5 it will probably have to come clean. Tell me what broke on your end in the comments.

Sources:
[Kotlin 2.5.0-Beta2 release notes](https://kotlinlang.org/docs/whatsnew-eap.html)
[Kotlin release schedule](https://kotlinlang.org/docs/releases.html)
[A more reliable compilation scheme for Kotlin Multiplatform modules](https://blog.jetbrains.com/kotlin/2026/09/a-more-reliable-compilation-scheme-for-kotlin-multiplatform-modules/)
