---
title: "One Piece of C Code, Five Pitfalls Each for Flutter and KMP"
date: 2026-11-13T08:11:41+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "Flutter", "FFI"]
summary: "34 lines of C bound twice, through Flutter's ffigen and KMP's cinterop. Under AOT both call in near 0.1us. KMP's debug build made struct returns 30x slower until release fixed it. Five pitfalls each, led by a version wall and a silently empty binding file."
cover:
    image: "/images/interop/cover.png"
    alt: "One Piece of C Code, Five Pitfalls Each for Flutter and KMP"
    relative: false
---

The Flutter team published an announcement on November 12, 2026, written by Mariam Hasnany, saying that FFIgen's Obj-C support and JNIgen are both out of preview. The post notes that Pigeon has been built on both mechanisms all along, and that this was simply the first time they said so out loud.

I went to try it that same afternoon.

I got stuck on step one: the latest ffigen would not install in my project.

Nobody at Flutter got anything wrong. Out of preview means the approach is mature. Whether the Flutter version on my machine can install the newest package is a separate question. Stack the two together, though, and you get the first wall everyone hits on day one.

## Stable in the announcement, errors at install

My machine runs Flutter 3.41.4 and Dart 3.11.1.

I put ffigen 23.0.0 in the pubspec and `pub get` failed outright, dumping a long dependency derivation: anything past ffigen 22 depends on code_assets, code_assets depends on hooks, hooks depends on record_use, record_use wants meta 1.19.0 or newer, and the Flutter SDK pins meta at 1.17.0. Six packages in a chain, broken at the fourth link, a full screen of errors, and the downgrade suggestion only on the last line.

That is where the chain snaps. The suggestion: install ffigen 21.

I changed the version to 21 and it went through immediately.

The version page on pub.dev lists 23.0.0, published October 6. Installable and newest are two different things.

So the first question is not how to use it. It is whether you can install it at all.

## So drop a version

Whether ffigen 21 covers everything the announcement describes, I cannot tell from a version number. The official write-up is about the design trade-offs of the approach; nowhere does it state a minimum version.

But what I wanted to measure needs neither JNIgen nor Obj-C bindings. I wanted the most basic case there is: one piece of C, bound twice, one per side, to see which is less work.

C is enough. C is the only intersection where both sides agree.

## Start with C both sides can use

I wrote 34 lines of C plus a 17-line header: four functions, deliberately in four different data shapes.

One is a pure computation: take an array, return the sum. This is the one that measures call overhead.

One returns a struct, and the struct carries a 64-byte inline char array. This is the shape that breaks things.

One writes a string into a buffer the caller hands over. Who allocates memory is exactly where the two sides disagree.

One returns a `const char` pointer.

All four ran at the C layer first, printing `sum=10`, `value=-7 negative`, `filled:42`, `interop-lab 1.0`. After both sides were bound, they had to reproduce that same set of values, not one character off.

![One piece of C, two binding paths](/images/interop/fig1.png)

## Flutter: one yaml gives you every binding

On the Flutter side I wrote 15 lines of config and 34 lines of calling code.

The config is a single yaml: where the header is, which dart file to write, which functions and which structs to include.

It ran in 0.76 s and produced 457 lines of Dart.

Then `dart run`, and all four calls came back correct.

One detail here is worth stopping on.

Of those 457 lines, only 23 have anything to do with my interface. The other 434 are fallout from the Darwin system headers: my header includes `stdint.h`, ffigen expands all the way down, and out come `pthread`, `mbstate_t` and a long tail of internal structures they drag in.

Readable. Not much use.

![Handwritten lines versus generated code](/images/interop/fig2.png)

## KMP: a four-line def plus one task

The KMP side takes three things.

A def file, four lines: header path, compiler options, linker options, package name.

A gradle config, twenty-two lines: define the macosArm64 target, create the cinterop, point it at the def file.

And four `expect` declarations in commonMain with four matching `actual` implementations in macosArm64Main, forty-two lines.

cinterop generates in 1.4 s. Linking the executable takes 2.8 s in debug and 5.7 s in release.

The output matched the C layer and the Dart side exactly.

Four functions, three implementations, one set of outputs. Until that lines up, comparing anything else is meaningless. If the values do not even agree, timing the builds is a waste of time.

## Build: KMP is four times slower and ten times smaller

Generating bindings: Flutter 0.76 s, KMP 1.4 s.

Compiling to something runnable: Dart AOT 1.3 s, KMP release 5.7 s. KMP is more than four times slower.

The artifacts flip it. The executable Dart compiles is 5.3 MB; the KMP kexe is 532 KB.

Ten times. Not because one is better optimized, but because of whether the runtime ships inside. The Dart binary carries the entire Dart runtime; KMP carries only the parts it uses.

The KMP debug build is 1.3 MB, also small, but that is because nothing was optimized, not because it is elegantly compact.

![Build time and artifact size](/images/interop/fig3.png)

## Runtime: even on plain calls, wide apart on structs

Twenty thousand runs each, averaged.

Plain function calls: Dart 0.095us, KMP 0.102us, a 7% gap that is basically noise. Struct returns: Dart 0.146us, KMP 0.349us, a 2.4x gap.

All four call shapes in one pass: Dart 56us, KMP 129us.

KMP starts faster, 22 ms across three runs. Dart gave 25, 25, 29.

So the gap is not global. It only shows up in a particular shape: pass an array, pass a pointer, take a string, and the two are equally fast. Make it return a struct by value and Kotlin pays for one extra CValue copy.

![Call overhead under release builds](/images/interop/fig4.png)

## A debug build nearly made me write the wrong conclusion

I nearly got this wrong.

My first KMP measurement used a debug build. Struct returns came in at 9.817us against Dart's 0.146us on the same case, and my draft already said "67 times slower."

Then I thought about it: debug does not optimize, so holding it against AOT is not a valid comparison. I rebuilt in release and the same case dropped to 0.349us.

A 28x gap, all of it hanging on one optimization flag.

Had I published that, the conclusion would have come out exactly backwards, and it would have looked very convincing.

## Five pitfalls against five

Five on the Flutter side.

One: the latest ffigen will not install, pinned out by the meta version, so you fall back to 21.

Two: write the entry-points in the config as a relative path and ffigen does not complain — it silently emits an empty bindings file. I saw a 229 B output at first and assumed my config was wrong. Absolute paths fixed it.

Three: for the inline char array inside a struct, ffigen generates an Array type with no cast method, so pulling the string out means reading it byte by byte.

Four: Char and Utf8 in the ffi package are two different extension types, and you have to convert back and forth between them.

Five: 434 of those 457 generated lines have nothing to do with you.

Five on the KMP side too.

One: on a first build the cinterop and compile tasks collide, so the first compile always fails with symbols not found. The second one works.

Two: the whole file has to opt in to ExperimentalForeignApi, and one annotation at the top of the file beats marking each method.

Three: ByteVar and IntVar live in kotlinx.cinterop, not in platform.posix. My first version imported them from posix and everything went red.

Four: a struct returned by value is a CValue in Kotlin/Native. You cannot touch its fields directly; you need useContents.

Five: after linking, the run fails because dyld cannot find the dylib. You need DYLD_LIBRARY_PATH, or an rpath of your own.

![Five pitfalls on each side](/images/interop/fig5.png)

## How the two sets of pitfalls differ

Same count, different nature.

Two of the Flutter five are silent: an empty bindings file raises nothing, and the version wall only stops you once, at `pub get`. You finish the code, it does not run, and you are left guessing what came out empty. All five KMP pitfalls are compile errors. You cannot dodge any of them, but you do not have to guess either: the errors are loud, and loud errors are faster to fix. The difference comes from when each side resolves its binding, one looking up symbols at runtime, the other fixing them at link time.

If what frightens you is a program that raises nothing and returns the wrong answer, the Flutter pitfalls hurt more. If what you dread is churning through builds, KMP costs you more.

## Which one to pick

One target, inside the Apple ecosystem: KMP's cinterop is less work. Bindings are fixed at compile time, the artifact is ten times smaller, and runtime cost is no worse.

Go cross-platform, especially to Android, Windows or JS, and the cinterop path ends. It exists only on targets Kotlin/Native supports; the moment you land on the JVM you are on the JNI stack instead.

ffigen, by contrast, is general: the same dylib, so or dll, only the load path changes with the platform.

This is the one thing I most want to say after all of it: these two are not substitutes at the same layer. cinterop is a compiler capability of Kotlin/Native; ffigen is a cross-platform binding generator. Racing them on speed is beside the point. What you are comparing is which path your project's shape leaves open.

As for that version wall, the day you want to try the newest ffigen, check your Flutter version first. Do not write the pubspec and only then discover it will not install, the way I did.
