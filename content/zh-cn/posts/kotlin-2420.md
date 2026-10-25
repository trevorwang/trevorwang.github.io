---
title: "Kotlin 2.4.20 发布了：这次 KMP 的 iOS 互操作，真的能打了"
date: 2026-09-18T07:02:39+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin"]
summary: "9 月 7 日，JetBrains 发布了 Kotlin 2.4.20。"
cover:
    image: "/images/kotlin-2420/cover.png"
    alt: "Kotlin 2.4.20 发布了：这次 KMP 的 iOS 互操作，真的能打了"
    relative: false
---

## Swift 互操作：从"能跑"到"好用"

9 月 7 日，JetBrains 发布了 Kotlin 2.4.20。

如果你只是扫一眼更新日志，可能会觉得"哦，又是个修修补补的版本"——毕竟两百多条变更里，大半是 bugfix。

但对写 KMP 的人来说，这个版本有几个东西是真香的。尤其是 Swift 互操作这块，终于补上了长期以来的几块短板。

先说结论：**这次最值得关注的是三个方向——Swift 互操作能力、编译速度、跨平台一致性**。下面一个个拆。

---

## 一、Swift export 支持 sealed class：iOS 同学的 switch 终于不用写 default 了

用 KMP 写过 iOS 的人，大概率都遇到过这个别扭的场景。

你在 Kotlin 里定义了一个很优雅的 sealed interface：

```kotlin
sealed interface Shape

class Circle : Shape
class Rectangle : Shape

fun createCircle(): Shape = Circle()
```

逻辑上天衣无缝，Kotlin 侧 pattern matching 一把梭。

但到了 Swift 那边，Kotlin 的 sealed 类型只能当成一个普通类来用，你根本没法做穷尽匹配，只能写一堆 `if` 判断，最后再兜一个 `default`。

更难受的是：**哪天 Kotlin 侧加了新的子类，Swift 侧的编译不会报错**——于是线上就可能出现一个没人处理的 case。

2.4.20 把这个问题解决了。sealed 类型现在会映射成 Swift 的 enum，编译器会生成一个 `sealedType()` 方法：

```swift
let shape = createCircle()

let name = switch shape.sealedType() {
case let .circle(type): "It's a \(type.value)"
case let .rectangle(type): "It's a \(type.value)"
}
```

注意这里——**没有 default**。

Swift 编译器会检查穷尽性，而且因为映射成了 enum，Xcode 的自动补全也跟着好用了。哪天 Kotlin 侧加了新的 sealed 子类，Swift 侧会直接编译报错提醒你处理。

这比"运行时才发现漏了 case"强太多了。

## 二、跨语言继承：用纯 Swift 库，不用再绕路了

第二个变化更实际：Swift export 支持跨语言继承了。

什么意思？说个真实场景——你想在 iOS 侧用 CryptoKit 这个纯 Swift 库，但它没法直接导入 Kotlin。

以前的做法要么是写一层 Objective-C 桥接，要么干脆放弃，在 Kotlin 侧找个替代实现。现在可以直接这么干：

Kotlin 侧声明接口和一个开放基类：

```kotlin
interface CryptoProvider {
    fun hashMD5(input: String): String
}

fun processHash(provider: CryptoProvider, input: String): String =
    provider.hashMD5(input)

open class SwiftBase
```

Swift 侧继承基类、实现接口，然后把对象传回 Kotlin：

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

Kotlin 收到这个 Swift 对象时，就把它当成接口的普通实现来调用。

这就是官方说的 reverse import 模式：**契约写在 Kotlin，实现在 Swift**。对那些必须依赖平台原生库的场景，这条路终于顺了。

## 三、增量编译进 Beta：改一行代码，不用再等一杯咖啡

这条可能是最能改善日常开发体验的。

Kotlin/Native 的编译速度一直是 KMP 的痛点之一。klib 的增量编译其实从 1.9.20 就有了，但一直在打磨。这次正式进入 Beta，官方说 debug 构建的编译时间大幅减少。

打开方式很简单，在 `gradle.properties` 里加一行：

```properties
kotlin.incremental.native=true
```

官方还提到，他们计划在后续版本里默认开启这个选项。

顺带一提，导出 XCFramework 的时候，`assembleSharedXCFramework` 任务现在会自动生成 `Package.swift` 文件。以前发布带 SwiftPM 依赖的 XCFramework，得自己处理 SwiftPM 包的分发问题，现在省事了。

---

## 四、Wasm 侧：几个不容易注意但很关键的改动

如果你的项目有 web 端，这几点值得留意。

**伴生对象初始化顺序对齐 JVM**

以前 Kotlin/Wasm 里，父类和子类的 companion object 初始化顺序可能和 JVM 相反。跨平台项目里，这种差异最容易变成"本地跑得好好的，换个平台就出鬼"的 bug。现在统一按 JVM 的行为来了。

**Wasmtime 运行时支持**

`wasmWasi` 目标以前只能用 Node.js 跑，还得先做一层 JavaScript 引导。现在可以直接用 Wasmtime 这个独立的 WebAssembly 运行时：

```kotlin
kotlin {
    wasmWasi {
        wasmtime()
    }
}
```

**新增编译模式**

以前只有 monolith 模式（整个项目和依赖一起编译，生成单个二进制，体积最小但每次都全量）。现在多了两个多模块模式，可以按需选择：

- 开发时用 `multimodule-closed-world`，只重编译改动的模块，迭代更快
- 生产构建仍用 monolith，保证产物最小

配置方式：

```properties
kotlin.wasm.compilationMode=multimodule-open-world
```

**注意一个 breaking change**：`@JsFun` 声明里现在用顶层 `require()` 会直接报错。如果之前这么写过，得迁移到 `@JsModule` 注解或者 `import()` 表达式。

## 五、Kotlin/JS：Karma 退役，Playwright 接棒

Kotlin/JS 的浏览器测试一直用 Karma 当启动器，但 Karma 项目已经停止维护两年了。

2.4.20 引入了新的测试 DSL：底层换成 Playwright 驱动（支持 Chromium、Firefox、WebKit 三家），测试运行器用 Mocha，打包工具暂时还是 webpack（官方说后续会换 Vite）。

用法是在 `browser {}` 里加一个 `test {}` 块：

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

另外，suspend lambda 现在可以导出成 JavaScript 的 async 函数了——这个对跨端调用挺实用。

## 六、标准库和其他

几个顺手能用的：

- **协程堆栈恢复**：新增 `StackTraceRecoverable` 接口。协程之间抛异常时，能看清异常从哪来、被谁重新抛出。自带复杂构造参数的异常，实现一下 `copyForStackTraceRecovery()` 就行。
- **集合工具**：新增了检查集合元素相等性和唯一性的函数。
- **kotlin.test**：新增断言函数重载。
- **Gradle**：完全兼容 Gradle 7.6.3 到 9.7.0；Problems API 现在带编译器诊断 ID 并做了分组，排错时能快速定位问题来源。
- **编译器**：JVM 21+ 上 `when` 表达式的 invokedynamic 编译从实验转正（默认开启）；新增 `kotlinr` 运行命令和独立的 native image。

---

## 这些变化对 KMP 开发者意味着什么

把上面串起来看，我觉得 2.4.20 释放了三个信号：

**1. iOS 互操作正在从"能用"走向"好用"**

sealed class 映射成 enum、跨语言继承支持纯 Swift 库——这两条直击 KMP 在 iOS 侧最常被吐槽的点。以前大家说 KMP "Android 体验好，iOS 差点意思"，现在这个差距在肉眼可见地缩小。

**2. 编译体验被当成正经事在解决**

klib 增量编译进 Beta、Wasm 多模块编译模式，都是在解决同一件事：**让日常开发不用等**。编译速度是 KMP 最现实的劝退点之一，这方向对了。

**3. 跨平台一致性越来越被重视**

Wasm 的伴生对象初始化顺序对齐 JVM，看起来是个小改动，但这类"平台间行为不一致"恰恰是跨平台项目最难缠的问题。官方在主动收窄这些差异。

顺带提一句另一条新闻：Google 前几天发布了 **ADK for Kotlin 1.0**，可以用 Kotlin 写生产级 AI Agent。Kotlin 和 AI Agent 这两个圈子，今年的交集肉眼可见地多起来了。

---

## 升级建议

- 一般项目：直接升。这个版本以修复为主，风险不大。
- **KMP 项目**：强烈建议升，Swift export 那几个改动是实打实的体验提升。
- **用 Wasm 的项目**：升级前先检查 `@JsFun` 里的 `require()` 调用，这是唯一的 breaking change。
- **想试增量编译**：先加 `kotlin.incremental.native=true` 试试水，有问题及时反馈给官方。

Kotlin 的版本节奏一直很稳，不搞大跃进。这种"每次解决几个真问题"的迭代方式，其实对生产项目最友好。

---

*参考资料：[What's new in Kotlin 2.4.20](https://kotlinlang.org/docs/whatsnew2420.html) · [GitHub Release](https://github.com/JetBrains/kotlin/releases/tag/v2.4.20)*
