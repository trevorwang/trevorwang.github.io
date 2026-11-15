---
title: "Kotlin 2.5 Value Classes Save No Memory, and Destructuring Changes Meaning"
date: 2026-11-13T08:20:14+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "Value Class", "Bytecode"]
summary: "Kotlin 2.5.0-RC pushes full value classes, so I read bytecode: javap shows a plain class, no box-impl. In a list it allocates 36.0 B, the same as the data class. The real win is a class file 21% smaller, and default destructuring still silently swaps names."
cover:
    image: "/images/kotlin25rc/cover.png"
    alt: "Kotlin 2.5 Value Classes Save No Memory, and Destructuring Changes Meaning"
    relative: false
---

On November 11, 2026, Kotlin 2.5.0-RC landed, and the headline feature is full value classes. A value class with several fields, which sounds exactly like a zero-cost abstraction.

Last month, when Beta2 came out, I wrote up a changelog round covering release formats and a few changed defaults. This time I wanted to measure something I skipped last time: does it actually save memory?

After installing the compiler, the first thing I did was not write a demo from the docs. It was run javap and read what the compiler emitted.

The conclusion first: in the bytecode it is an ordinary class. Not a trace of inline anywhere.

## First, confirm what the compiler accepts

Download kotlin-compiler-2.5.0-RC.zip, unzip it, and the version is kotlinc-jvm 2.5.0-RC, running on JRE 21.0.10.

The first step hit a wall. Write a `value class VComplex(val real: Double, val imaginary: Double)` and the compiler refuses:

the feature "full value classes" is experimental and should be enabled explicitly.

You need `-Xfull-value-classes` before it accepts anything. The docs say the feature is Experimental and must be enabled explicitly. All true, it just does not tell you what you will see once it is enabled.

![2.5.0-RC features and flags](/images/kotlin25rc/fig1.png)

## Bytecode is more honest than the docs

I wrote two controls. One `data class DComplex(real, imaginary)`, one `value class VComplex(real, imaginary)`. Same fields, same types.

`javap -p` on VComplex: a private final double real, a private final double imaginary, a two-argument constructor, two getters, one toString.

That is all. No `box-impl`, no `unbox-impl`, and no packing of the two doubles into some other representation.

Now the single-field `@JvmInline value class VUserId(val raw: Long)`, which looks completely different: `constructor-impl(long)`, `box-impl(long)`, `unbox-impl()`, `hashCode-impl(long)`, `equals-impl(long, Object)`, and every one of those methods takes a primitive long.

That is what inline means. The multi-field one is not it.

![Bytecode comparison](/images/kotlin25rc/fig2.png)

## So does it save anything?

I measured allocation with `ThreadMXBean.getCurrentThreadAllocatedBytes`, 1,000,000 runs, looking at average bytes per run.

First round, reading fields off local variables: VComplex 9.3 B, DComplex 10.4 B. Both numbers are far below what an object should occupy, because JIT did scalar replacement and the object was never created at all.

That measurement does not hold up. So I moved them into an ArrayList, forcing the objects to exist.

VComplex into a List: 36.0 B per run. DComplex into a List: 36.0 B per run.

Identical. Zero bytes apart.

The 36 is easy enough to explain: a 16-byte object header plus 16 bytes for the two doubles. A clean integer.

So on the JVM, a multi-field value class saves not a single byte. The only size difference from the data class is in the class file: VComplex.class is 1,760 bytes, DComplex.class is 2,216 bytes, 21% smaller, and what is missing is copy and componentN.

## Not even @JvmInline escapes

I measured the single-field control while I was at it. VUserId into a List: 28.0 B. DUserId as well: 28.0 B.

Again not a byte apart.

The reason is plain: once it goes into a generic container it has to be boxed, and once boxed it is an object, indistinguishable from an ordinary class.

So the claim that an inline value class is a zero-cost abstraction holds only at the layer of passing arguments directly. Put it inside a `List`, a `Map`, an `Array<Any>`, and the cost comes straight back.

This is not a 2.5 problem, it is the old rule of JVM generic erasure. It is just that a lot of people reach for value classes as a memory-saving tool.

![Allocated bytes comparison](/images/kotlin25rc/fig3.png)

## What it saves is semantics

So what is a multi-field value class for?

It is for the idea that two instances with identical contents should not differ by identity. A data class `==` compares contents, but two instances are still two objects, each with its own reference, its own lock, its own place on the heap.

A value class removes that identity. You cannot rely on reference equality, which means you cannot use it as a lock and cannot use object identity as a cache key.

The price is no `copy()` and no `componentN()`. The docs mention copy vars coming later. Not yet.

I did measure deduplication and equals: `setOf(a, b).size` is 1 both ways, and `toString` prints `VComplex(real=1.0, imaginary=2.0)`, the same as the data class.

So what it gives you is a constraint, not performance.

## Where it really differs from a data class

I measured the equals edge cases too, because that is where surprises like to hide.

NaN against NaN: true for VComplex, true for DComplex. Negative zero against positive zero: false on both sides. Same behaviour as a data class, all of it following Double.compare semantics. No extra surprise, no extra trap.

Inheritance does have something new. A `value class` can implement a sealed interface, and a `value object` can join in:

```kotlin
sealed interface Resp
value class Found(val id: Int) : Resp
value object Unknown : Resp
```

`Unknown::class.java.name` prints Unknown, and the compiler resolves `Found(1) is Resp` to true outright, with a warning that the check is always true. Because a value class's inheritance is fixed at compile time, there is no second possibility at runtime.

So the substantive differences from a data class are three: no `copy()`, no `componentN()`, no object identity.

The first two are subtractions. The third is the reason it exists.

## Destructuring has a silent trap

The RC made only-syntax the default mode for name-based destructuring. I wrote about the trap last time, so here is harder evidence.

`data class User(val username: String, val email: String)`, an instance of `User("alice", "alice@example.com")`, and then one line: `val (email, username) = user`.

The variable names and the property names are swapped. This is extremely common in production code. You type by position, not by name.

Default mode prints: email is alice, username is alice@example.com. Completely transposed.

Add `-Xname-based-destructuring=complete`, recompile the same source, not one character changed, and it prints: email is alice@example.com, username is alice.

Both modes compile. Neither reports an error. The runtime results are exact opposites.

There is a name-mismatch mode in between that does not change behaviour, only warns, and its exact wording is variable name 'email' differs from accessed property name 'username', this code will change its meaning.

It does not say this is a bug. It says the meaning of this code will change. If you want to clear the mines early, this is the only mode you can use.

![Same code, two results](/images/kotlin25rc/fig4.png)

## The migration flags are still missing in RC

The docs, in the migration section, say: starting with 2.5.20, unmarked single-field value classes produce a warning, 2.6 or 2.7 promotes that to an error, and until then you can use `-Xinline-value-class-migration=warning` and `strict` to prepare.

I tried exactly that. The compiler replied: flag is not supported by this version of the compiler. Neither one is accepted.

All three destructuring modes work, not one of the two migration flags exists. The docs are not wrong, they describe a state that is coming, but written on the RC page they read as available now.

![What the docs say vs what RC does](/images/kotlin25rc/fig5.png)

## Verify it yourself in two minutes

Do not take my word for it. The commands are below.

Download and unzip the compiler, write a minimal comparison file, compile with the flag, then open both classes with javap. Look for the `-impl` suffix and you know at a glance whether it is inline.

The allocation measurement is simple too, `ThreadMXBean` in two lines:

```kotlin
val mx = ManagementFactory.getThreadMXBean() as com.sun.management.ThreadMXBean
val before = mx.getCurrentThreadAllocatedBytes()
// 这里跑你的循环
val after = mx.getCurrentThreadAllocatedBytes()
println((after - before) / 循环次数)
```

One thing you have to watch: the local-variable scenario measures nothing. JIT eliminates the object entirely and both numbers come out at a dozen bytes or so. Put it in a collection and measure that, because that is what it looks like when the object is really created.

I fell into this on my first attempt and nearly wrote down the wrong conclusion, that both sides save memory.

## What you can do before upgrading

Three things.

First, hang `-Xname-based-destructuring=name-mismatch` on your build and run it once. Zero behaviour change, pure checking, and the warnings it produces are the line numbers whose meaning will change later.

Second, if you are using multi-field value classes, work out whether you are after memory or after semantics. If it is memory, the measurements say you will not get it.

Third, do not put value classes on your list of performance optimizations. True zero cost only holds for single-field direct argument passing, and you have to write `@JvmInline` explicitly, because from 2.5 on the words `value class` no longer automatically mean inline.

As for full value classes themselves: experimental, gone the moment the flag is off, and production use will have to wait.

The one thing I most want to say after all this measuring: a type with value in its name is still an object on the JVM. If you want to save memory, you still have to look at where your data structures actually live.
