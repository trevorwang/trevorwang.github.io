---
title: "Kotlin 2.4.20 Is Out: KMP's iOS Interop Can Finally Hold Its Own"
date: 2026-09-18T07:02:39+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "On September 7, JetBrains released Kotlin 2.4.20."
cover:
    image: "/images/kotlin-2420/cover.png"
    alt: "Kotlin 2.4.20 Is Out: KMP's iOS Interop Can Finally Hold Its Own"
    relative: false
---

## Swift interop: from "it runs" to "it is good to use"

On September 7, JetBrains released Kotlin 2.4.20.

If you only skim the changelog you might think, oh, another patch-and-polish release — after all, of the two hundred-odd changes, most are bugfixes.

But for people writing KMP, a few things in this version are genuinely nice. Swift interop especially finally closes several long-standing gaps.

Conclusion first: **the three things worth watching are Swift interop, compile speed, and cross-platform consistency**. Taking them one at a time.

---

## 1. Swift export supports sealed class: iOS folks can finally skip default in their switch

If you have written iOS with KMP, you have almost certainly hit this awkward spot.

You define a perfectly elegant sealed interface in Kotlin:

```kotlin
sealed interface Shape

class Circle : Shape
class Rectangle : Shape

fun createCircle(): Shape = Circle()
```

Logically seamless. On the Kotlin side you pattern match in one shot.

But on the Swift side, a Kotlin sealed type could only be treated as an ordinary class. You could not do exhaustive matching at all — just a pile of `if` checks and a `default` at the end to catch the rest.

The worse part: **the day someone adds a new subclass on the Kotlin side, the Swift side does not fail to compile** — so a case nobody handles can reach production.

2.4.20 fixes this. Sealed types now map to Swift enums, and the compiler generates a `sealedType()` method:

```swift
let shape = createCircle()

let name = switch shape.sealedType() {
case let .circle(type): "It's a \(type.value)"
case let .rectangle(type): "It's a \(type.value)"
}
```

Note this — **no default**.

The Swift compiler checks exhaustiveness, and because it maps to an enum, Xcode autocomplete works properly too. The day a new sealed subclass appears on the Kotlin side, the Swift side fails to compile and tells you to handle it.

That is a lot better than "finding out at runtime that a case is missing."

## 2. Cross-language inheritance: use pure Swift libraries without the detour

The second change is more practical: Swift export now supports cross-language inheritance.

What does that mean? A real scenario — you want to use CryptoKit, a pure Swift library, on the iOS side, but it cannot be imported into Kotlin directly.

The old approach was either an Objective-C bridging layer, or giving up and finding a substitute implementation on the Kotlin side. Now you can just do this:

Declare an interface and an open base class on the Kotlin side:

```kotlin
interface CryptoProvider {
    fun hashMD5(input: String): String
}

fun processHash(provider: CryptoProvider, input: String): String =
    provider.hashMD5(input)

open class SwiftBase
```

Inherit the base class and implement the interface on the Swift side, then pass the object back to Kotlin:

```swift
import CryptoKit

final class IosCryptoProvider: SwiftBase, CryptoProvider {
    func hashMD5(input: String) -> String {
        guard let data = input.data(using: .utf8) else { return "failed" }
        return Insecure.MD5.hash(data: data).description
    }
}

let provider = IosCryptoProvider()
print(processHash(provider: provider, input: "Hello, world!"))
```

When Kotlin receives that Swift object it calls it as a normal implementation of the interface.

This is what the team calls the reverse import pattern: **the contract lives in Kotlin, the implementation lives in Swift**. For cases where you must depend on a platform-native library, the road is finally clear.

## 3. Incremental compilation hits Beta: change one line, stop waiting for a coffee

This one may be the biggest improvement to day-to-day development.

Kotlin/Native compile times have always been one of KMP's sore spots. klib incremental compilation has existed since 1.9.20, but it kept getting polished. Now it enters Beta officially, and the team says debug build times drop substantially.

Turning it on is one line in `gradle.properties`:

```properties
kotlin.incremental.native=true
```

The team also mentions they plan to enable it by default in an upcoming version.

While we are here: when exporting an XCFramework, the `assembleSharedXCFramework` task now generates a `Package.swift` file automatically. Publishing an XCFramework with SwiftPM dependencies used to mean handling SwiftPM distribution yourself. That is now taken care of.

---

## 4. Wasm: a few changes that are easy to miss and hard to skip

If your project has a web side, these are worth noting.

**Companion object initialization order now matches the JVM**

In Kotlin/Wasm, companion object initialization order for parent and child classes could run opposite to the JVM. In cross-platform projects, that kind of difference turns into the worst bugs: works fine locally, behaves like a ghost on another platform. It now follows JVM behavior.

**Wasmtime runtime support**

The `wasmWasi` target used to run only through Node.js, and needed a JavaScript bootstrap layer. Now you can use Wasmtime, the standalone WebAssembly runtime:

```kotlin
kotlin {
    wasmWasi {
        wasmtime()
    }
}
```

**New compilation modes**

Previously there was only monolith mode (compile the whole project and its dependencies together into a single binary — smallest output, but a full rebuild every time). Now there are two multi-module modes, so you can pick:

- Use `multimodule-closed-world` during development to recompile only changed modules, for faster iteration
- Keep monolith for production builds, to keep the output as small as possible

Configuration:

```properties
kotlin.wasm.compilationMode=multimodule-open-world
```

**Watch for one breaking change**: top-level `require()` inside a `@JsFun` declaration now fails to compile. If you wrote it that way before, migrate to the `@JsModule` annotation or an `import()` expression.

## 5. Kotlin/JS: Karma retires, Playwright takes over

Kotlin/JS browser tests have always used Karma as the launcher, but the Karma project has gone unmaintained for two years.

2.4.20 introduces a new test DSL: Playwright underneath as the driver (supporting Chromium, Firefox, and WebKit), Mocha as the test runner, and webpack still as the bundler for now (the team says Vite comes later).

Usage is a `test {}` block inside `browser {}`:

```kotlin
kotlin {
    js {
        browser {
            @OptIn(ExperimentalJsTestDsl::class)
            test {
                timeout = 2.seconds
                chromium()
                firefox()
                webkit()
            }
        }
    }
}
```

Also, suspend lambdas can now be exported as JavaScript async functions — handy for cross-platform calls.

## 6. Standard library and everything else

A few handy ones:

- **Coroutine stack trace recovery**: a new `StackTraceRecoverable` interface. When exceptions fly between coroutines, you can see where they came from and who rethrew them. For exceptions with complex constructor parameters, implement `copyForStackTraceRecovery()`.
- **Collection utilities**: new functions for checking element equality and uniqueness.
- **kotlin.test**: new assertion function overloads.
- **Gradle**: full compatibility from Gradle 7.6.3 through 9.7.0; the Problems API now carries compiler diagnostic IDs and groups them, so you can locate the source of a problem fast when debugging.
- **Compiler**: invokedynamic compilation of `when` expressions on JVM 21+ is now stable (on by default); a new `kotlinr` run command and a standalone native image.

---

## What these changes mean for KMP developers

Put it all together and I think 2.4.20 sends three signals.

**1. iOS interop is moving from "works" to "works well"**

Sealed classes mapping to enums, cross-language inheritance for pure Swift libraries — both hit exactly what people complain about most on the iOS side of KMP. People used to say KMP is "good on Android, a bit lacking on iOS." That gap is visibly narrowing.

**2. Compile experience is being treated as a real problem**

klib incremental compilation into Beta, Wasm multi-module compilation — both solve the same thing: **not making daily development wait**. Compile speed is one of KMP's most concrete reasons people bounce off it. This is the right direction.

**3. Cross-platform consistency is getting more attention**

Wasm companion object initialization matching the JVM looks like a small change, but "behavior differs across platforms" is exactly the nastiest class of bug in cross-platform projects. The team is actively narrowing those differences.

One other piece of news while we are here: Google released **ADK for Kotlin 1.0** a few days ago, for writing production-grade AI agents in Kotlin. The Kotlin and AI agent circles have visibly been overlapping a lot more this year.

---

## Upgrade advice

- Ordinary projects: just upgrade. This release is mostly fixes, low risk.
- **KMP projects**: strongly recommended. The Swift export changes are a real improvement in day-to-day feel.
- **Projects using Wasm**: check `require()` calls inside `@JsFun` before upgrading. That is the only breaking change.
- **Want to try incremental compilation**: add `kotlin.incremental.native=true` and test the waters; report problems upstream.

Kotlin's release cadence has always been steady, no great leaps. This "fix a few real problems each time" style of iteration is, honestly, the friendliest thing for production projects.

---

*References: [What's new in Kotlin 2.4.20](https://kotlinlang.org/docs/whatsnew2420.html) · [GitHub Release](https://github.com/JetBrains/kotlin/releases/tag/v2.4.20)*
