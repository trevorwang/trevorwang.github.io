---
title: "Kotlin 语法糖拆到字节码：哪几个真免费"
date: 2026-11-13T12:57:05+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin"]
summary: "Kotlin 有一半的便利是语法糖给的，这一点在 2026 年的 2.5.0-RC 上没变。扩展函数、中缀调用、操作符重载、空安全、默认参数、data class，写起来都像语言原生能力。"
cover:
    image: "/images/sugar/cover.png"
    alt: "Kotlin 语法糖拆到字节码：哪几个真免费"
    relative: false
---

Kotlin 有一半的便利是语法糖给的，这一点在 2026 年的 2.5.0-RC 上没变。扩展函数、中缀调用、操作符重载、空安全、默认参数、data class，写起来都像语言原生能力。

但 JVM 不认识这些。编译完它只认类、方法、字段、字节码指令。

我把常用的十几个语法糖挨个编译，用 javap 打开看它们变成了什么。分类很清楚：有三个是真免费，有几个只是省了打字，还有几个偷偷付了钱。

![十几个语法糖按成本分三档](/images/sugar/fig1.png)

## 先说最干净的那个

`s?.length ?: -1`，一个空安全调用加一个 elvis 兜底。

编译完只有 8 条指令：

aload 读参数，dup 复制一份，ifnull 跳到分支，调 length，goto 汇合，pop 掉多余的栈顶，推 -1，返回。

没有对象分配，没有额外的函数调用，没有 try catch。就是一个 ifnull 跳转。

这是真零成本。你写 `?.` 和手写 `if (s != null) s.length else -1` 出来的字节码基本一样。

## 但 !! 是要收费的

同样是让编译器相信我，`s!!.length` 编出来 5 条指令，中间夹着一个 `Intrinsics.checkNotNull` 静态调用。

每次执行到 `!!`，都会真的去做一次非空检查，失败就抛 NullPointerException。

这不是编译期的保证，是运行时的一次函数调用。

更有意思的是普通参数。我写了一个 `fun checked(s: String): Int = s.length`，参数是非空类型，什么都没做。编译完 6 条指令，第一件事是 `Intrinsics.checkNotNullParameter(s, "s")`。

也就是说，每个 Kotlin 函数的公开入口，只要参数标了非空，都会先检查一遍。`!!` 是显式检查，非空参数是隐式检查，两者在字节码里是同一类东西。

![空安全三种写法的代价](/images/sugar/fig2.png)

## 扩展函数和中缀调用：纯改写法

`fun String.shout(): String`，编译成一个静态方法 `shout(String)`，receiver 变成了第一个参数。

`infix fun Int.plus2(other: Int)`，也是一个静态方法 `plus2(int, int)`。

`operator fun plus(o: Op)`，就是一个普通成员方法 `plus(Op)`。

这三个在字节码里没有任何痕迹表明它们「曾经是语法糖」。调用处就是普通的 invokestatic 或 invokevirtual。

零成本，因为它们压根没有改变程序结构，只是换了一种写法描述同一个方法。

## when 用的是 tableswitch

`when (n) { 0 -> ...; 1 -> ...; 2 -> ...; else -> ... }`，如果分支值是连续的整数，编出来是 `tableswitch`。

这是 JVM 里最快的那种分支，一次查表跳转，不需要逐个比较。

如果分支值稀疏，会退化成 `lookupswitch`，本质是二分查找。如果分支不是常量，就变成一串 if 判断。

所以 when 的性能完全取决于你写什么值，编译器不保证最优。写连续常量它给你 tableswitch，写别的它降级。

## data class：十个方法

一个 `data class Point(val x: Int, val y: Int)`，class 文件 2111 字节，生成十样东西：

两个字段、一个构造器、两个 getter、component1、component2、copy、copy$default、toString、hashCode、equals。

你在源码里写了一行，字节码里是一整个类。

这十样里，component1 和 component2 就是为了解构服务的。你写 `val (a, b) = point`，编译后就是两次 componentN 调用。

copy$default 是为了带默认参数的 copy 服务，参数是 `(Point, int, int, int, Object)`，第三个 int 是位掩码，告诉你哪几个参数用了默认值。

![一行 data class 生成的十个东西](/images/sugar/fig4.png)

## object：一个静态字段

`object Singleton` 编出来是：

一个 `public static final Singleton INSTANCE` 字段，一个私有构造器，一个静态初始化块。

所谓单例，就是一个类把自己锁死，然后用静态字段持有唯一实例。你在 Kotlin 里写 `Singleton.name`，字节码里是 `getstatic INSTANCE` 加一次方法调用，没有反射，也没有延迟初始化那一套。

## 默认参数：多一个方法

`fun greet(name: String, title: String = "engineer")`，除了 `greet(String, String)`，还生成一个 `greet$default(String, String, int, Object)`。

那个 int 是位掩码，第几位是 1 就表示第几个参数用了默认值。Object 是给内联类预留的标记位。

调用点会判断：全都传了就走正常方法，漏一个就跳到 $default，由它补齐再转调。

我实测过传满和漏一个的耗时，各 500 万次，都在 8 到 9 毫秒之间，跟普通五参函数一样。这个开销量不出来。

## lazy 委托：一个字段加一个类

`val text: String by lazy { ... }` 编出来 1416 字节：

一个 `private final kotlin.Lazy text$delegate` 字段，一个 `getText()` 方法，还有一个静态方法装那个 lambda。

每次读 `text` 都是 `getText()`，里面转调 `Lazy.getValue()`。所以 lazy 的属性访问永远比普通属性多一次虚调用。

顺便说个反直觉的观察。文档提到 LazyThreadSafetyMode 有三种取值，我把 SYNCHRONIZED、PUBLICATION、NONE 各跑 500 万次，5 毫秒和 36 毫秒这种差别在多次运行之间会翻转，量不出稳定结论。这个层面 JIT 的行为比模式本身影响更大。

所以别照着博客选模式，除非你在自己的负载上真量过。

## inline：省的是对象，费的是代码

inline 这一项最有意思，因为它两头都占：省掉一次调用的同时，把代码复制了好几份。

我写一个 `inline fun inlined(x: Int)` 和内容完全一样的普通函数，各被六个函数调用。

普通版本，调用点只有 invokestatic，函数体在字节码里只存在一份。

inline 版本，六个调用点全把函数体复制了进来，每个 useX 方法里都能直接看到 imul 和 ishl 指令。

这是 inline 的本质：省掉一次函数调用和一个可能的 lambda 对象，代价是代码重复。

但省掉的到底有多少。我量了 100 万次带 lambda 的高阶函数调用：inline 版本分配 0 字节，普通版本分配 9872 字节。

一百万分之九千八，平均每次 0.01 字节，相对那个 36 字节的对象只有 0.03%。

因为 JIT 把这个 lambda 对象逃逸分析掉了。现代 JVM 上，非 inline 高阶函数的对象开销基本不存在。

更讽刺的是，我那个 inline 例子编译时报了一句警告，官方文档写道 inline 只对函数类型参数有意义，编译器把这句话直接甩在了编译输出里：

expected performance impact from inlining is insignificant. Inlining works best for functions with parameters of function types.

编译器在说：你这个 inline 没用，只有当参数是函数类型时才值得。

编译器自己在劝你别乱用 inline，这比大部分技术博客都诚实。

![inline 的代码复制](/images/sugar/fig3.png)

## 字符串模板还在用 StringBuilder

还剩一处，也是最意外的。

`"$name is $age years old, ${age * 2}"`，编出来是 `new StringBuilder` 加五次 append 加 toString。

而 Java 从 9 开始就已经用 invokedynamic 加 String Concat Factory 了，Kotlin 在这条路上慢了一拍。

我以为是开关的问题，试了 `-Xstring-concat=indy-with-constants`，结果编译报了个警告：this flag does nothing with JVM target 1.8。

原来 Kotlin 默认的 jvmTarget 就是 1.8，class 文件 major version 52。

四种组合我全试了：target 1.8 加不加开关都是 StringBuilder；target 21 不加开关就自动变成 invokedynamic。

决定因素是 jvmTarget，不是那个开关。

所以你写的字符串模板用哪种实现，取决于构建脚本里 `jvmTarget` 写的是多少。Android 项目大多还是 1.8 或 11，那就是 StringBuilder。

![四种组合四种实现](/images/sugar/fig5.png)

## 拆完之后的分级

拆完可以分成三档。

真免费：空安全调用、扩展函数、中缀调用、操作符重载。它们在字节码里就是普通的方法调用和跳转，一个字都不多。

只多代码不多时间：data class、object、默认参数、when。生成的东西看着吓人，但运行时的行为跟手写一样。

有真实成本：`!!` 和每个非空参数的运行时检查、lazy 的每次委托调用、inline 的代码复制、字符串模板在低 jvmTarget 下的 StringBuilder。

还有一档是「看配置」：字符串模板用哪种实现，你改一行 jvmTarget 就变了。

我拆完最大的感受是，语法糖这个词容易让人以为「反正编译器会处理好」。实际上编译器只是忠实地把你写的东西翻译成指令，好坏全看你写的是什么。
