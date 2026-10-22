---
layout: post
title: "Kotlin Multiplatform 编译\"看心情\"这事，官方认了"
date: 2026-10-03 22:24:18 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/kmp-sepcomp/cover.png
---

![Kotlin Multiplatform 编译"看心情"这事，官方认了](/assets/images/kmp-sepcomp/cover.png)

## 先说那个瞬间

你写 Kotlin Multiplatform 的时候大概遇到过：IDE 里那行代码红着，划红线说找不到符号，你犹豫半天不敢提交，结果一构建，绿的，跑起来还对。

以前我们把这事归给 IDE 抽风、索引没建好、清一下缓存。9 月 28 日 JetBrains 发了篇博客，开头第一句就直白得很：

> The current compilation approach to Kotlin Multiplatform projects works, but sometimes can lead to unexpected or hard-to-predict behavior.

翻译成人话：现在这套编译方式能跑，但有时候会给你难以预测的行为。然后它列了两个例子，都是我们遇到过但说不清原因的：

> The IDE analysis disagrees with the compiler on an overload, type inference, or whether the code should compile at all – with the IDE being stricter than the compiler.

> Your `commonTest` code can unexpectedly call something from a platform source set, breaking your assumptions.

一句是 IDE 比编译器严格，一句是 commonTest 会悄悄调到平台源集的东西。下面我先把机制讲清，再给我实测的结果，因为官方那个例子里，修法的代价只给你看了一半。

## 为什么会有这种分歧

关键在「common 代码编译时，是拿什么去解析依赖的」。

KMP 模块里，commonMain 不是独立编出来一个产物就完事。编 JVM target 的时候，编译器把 commonMain 和 jvmMain 一起，对着依赖的平台产物（JVM 上是 jar）来编。于是 commonMain 里的调用，有机会隐式落到依赖方 jvmMain 的声明上。

而 IDE 的代码分析不这么干。它假设 common 代码只能看见依赖的 common 声明，也就是 metadata KLIB 里列出来的那部分。jvmMain 的声明不在里面，所以 IDE 报红。

两边对同一行代码有两套理解，谁都不算错，但你会被夹在中间。官方还给了一个更难受的版本：重载选择不一致，IDE 跳转到 commonMain 的重载，运行时打印的却是平台那一个。

以及类型推断分歧，common 侧推出来是 Any，JVM 编译直接报 Unresolved reference。这条反过来也成立：同一个例子开着开关，common 阶段就把 currentEvent() 的返回类型定成 Event，`.name` 有依据，JVM 编译直接通过。所以这个特性两头都会发生，既会让侥幸能过的代码报错，也会把原来报错的代码修好。

上面这两条路径的差别，画出来是这样：

![](/assets/images/kmp-sepcomp/lookup-paths.png)

## 新方案就是一个开关

Kotlin 2.5.0-Beta1 引入了一个可选的「分离编译」（separate compilation），让编译器在编 common 源集时也走 metadata KLIB 那套严格解析，跟 IDE 对齐。

```properties
# gradle.properties
kotlin.kmp.separateCompilation=true
```

官方标了 Experimental，默认关闭。附带一个好处：它让 common 源集的增量编译成为可能，这是以前做不到的。

官方给的例子读起来很轻松，原来 IDE 报红、编译却能过的代码，现在编译也报错；把 expect/actual 显式连起来就好了。

## 实测一：它真的会拦下来

我在本机搭了个最小的 KMP 工程（Kotlin 2.5.0-Beta1，两个 target：jvm 和 js），两个模块：

```text
lib/  src/commonMain/Marker.kt      # 只有接口
      src/jvmMain/Foo.kt            # class Foo，只声明在平台源集
app/  src/commonMain/Main.kt        # 直接 new 一个 lib 的 Foo
```

app 依赖 lib，app 的 commonMain 里 new 一个只存在于 lib/jvmMain 的类。

不打开开关：

```text
> Task :app:compileKotlinJvm
BUILD SUCCESSFUL in 30s
```

打开开关，同一份代码：

```text
e: app/src/commonMain/kotlin/Main.kt:3:12 Unresolved reference 'Foo'.
e: app/src/commonMain/kotlin/Main.kt:6:13 Unresolved reference 'Foo'.
BUILD FAILED in 2s
```

官方说的没错，而且拦得很干脆，不是警告。

commonTest 那条也一样。我在 app 的 jvmMain 放了个 PlatformOnly 类，让 commonTest 去 new 它：关着开关 BUILD SUCCESSFUL，打开就是 `Unresolved reference 'PlatformOnly'`。

## 实测二：修法的代价，官方例子没给你看

官方的修法是显式 expect/actual。lib 的 commonMain 声明 expect，平台源集给 actual。我照着改，JVM 端确实过了：

```kotlin
// lib/commonMain
expect class Foo()

// lib/jvmMain
actual class Foo actual constructor()
```

开着开关，`:app:compileKotlinJvm` 是 BUILD SUCCESSFUL in 1s。这一半官方说得对。

然后我去编同一个模块的 JS 端：

```text
e: lib/commonMain/kotlin/Marker.kt:5:1 The 'expect' declaration 'lib.Foo'
   has no 'actual' declaration in module '<commonMain> for JS'.
BUILD FAILED in 2s
```

expect/actual 是成对契约，每个 target 都得补一个 actual。博客里的例子只演了 JVM 那一侧，你多平台项目里一开，有几个 target 就是几份 actual。

## 官方没提的一件事：版本号不是门槛

博客说这个特性是 Kotlin 2.5.0-Beta1 引入的。我拿同一份「坏代码」在 Kotlin 2.4.20（当前稳定版）上又试了一遍：

```text
# 不加这个属性
BUILD SUCCESSFUL in 704ms

# 加上 kotlin.kmp.separateCompilation=true
e: app/src/commonMain/kotlin/Main.kt:3:12 Unresolved reference 'Foo'.
BUILD FAILED in 11s
```

属性在 2.4.20 上照样切换行为，我核过这份工程确实解析到 kotlin-gradle-plugin:2.4.20。

所以这个开关不是 2.5 才存在的东西，只是官方到 2.5.0-Beta1 才把它当特性讲出来。反过来看：别把版本号当门槛，也别把「我 2.4 上试了没事」当结论，两边都不牢靠，得在你自己的工程上跑一遍才算数。

## 一张速查表

把上面这些现象和官方那节 Known issues 收成一张表，撞上对应情形时对着看：

| 旧代码的情况 | 新方案下的结果 | 怎么办 |
|---|---|---|
| commonMain 偷用了依赖库 jvmMain / nativeMain 的声明 | 编译失败，报 Unresolved reference | 让库把这项能力声明成 expect/actual，或者把调用下沉到平台源集 |
| commonTest 偷用了 jvmMain 等平台代码 | 测试编译失败 | 用 expect/actual 暴露公共接口，或者把测试搬到 jvmTest |
| common 与平台侧存在同名重载 | 选中的重载可能变（我实测：从 platform 变成 common） | 检查这类调用以前实际落到了哪个重载 |
| common 的类型推断受平台 actual 额外超类型影响 | 推断结果被固定住，可能顺手修掉旧报错，也可能暴露依赖旧行为的代码 | 给 common 侧显式声明类型边界 |

## 你的项目该不该开

结合官方那节 Known issues，我的判断清单是这样：

- 依赖的库有没有正经发 metadata KLIB。新方案让它从「最好有」变成「必须有」，作者偷懒的库一开就报错，而错不在你。
- 工程有多依赖「common 调平台」这种写法。我手上那个 KMP 项目平台差异走接口、几乎不写 expect/actual，开之前得先把这类调用扫一遍，否则会一次性冒出一堆 Unresolved reference。
- 用不用 cinterop。官方点明了这两处已知问题（KT-88178、KT-41509），原生 C 互操作的项目要谨慎。
- 只有一个 target 的模块不受影响，官方说得很直白。
- 它还是 Experimental，默认关着。当预览用，别当升级项。

## 顺带一句 iOS 那条战线

同一周 JetBrains 还发了篇 TeamCity 的设计复盘，讲怎么把 Kotlin Multiplatform 的 iOS 打包做成引导式流程：装个插件，push 之后 IDE 弹一个 Configure CI 的提示，跟着走就能从首次 push 到跑测试、签名、发 TestFlight，最后接上 GitHub 每次推送自动跑。

中间生成四个文件（`teamcity.yaml` 加 Fastfile、Appfile、Gemfile），先留在你本地不提交，第一次构建跑在托管的 macOS agent 上，验过再让你自己提交。

这块我没实测，本机没有 TeamCity 账号，那段内容全部来自官方那篇设计复盘和它指向的官方教程。

## 最后

回到开头那个 IDE 报红、编译能过的瞬间。它过去既不是 IDE 的 bug，也不是你的错觉，是两套解析规则长期没对齐。官方现在的做法是把编译器往 IDE 那边拉：更严格、更可预测，代价是以前侥幸能过的代码，要先补成显式的。

我下一步想在手上那个 KMP 项目里开一次，把冒出来的 Unresolved reference 全列出来，看看这个项目到底欠了多少 expect/actual。

你的 Kotlin Multiplatform 项目开过这个开关吗？开完报了多少错？评论区聊聊。

参考资料：
- A More Reliable Compilation Scheme for Kotlin Multiplatform Modules：https://blog.jetbrains.com/kotlin/2026/09/a-more-reliable-compilation-scheme-for-kotlin-multiplatform-modules/
- Designing the Kotlin Multiplatform and TeamCity Integration：https://blog.jetbrains.com/teamcity/2026/09/designing-the-kotlin-multiplatform-and-teamcity-integration/
- 官方 iOS CI/CD 教程：https://kotlinlang.org/docs/multiplatform/ios-ci-cd-teamcity.html
