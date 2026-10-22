---
layout: post
title: "AGENTS.md 来了，KMP 项目该怎么写这份说明书"
date: 2026-09-19 07:19:11 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/agents-md-kmp/cover.png
---

![AGENTS.md 来了，KMP 项目该怎么写这份说明书](/assets/images/agents-md-kmp/cover.png)

## 一场持续近一年的拉锯，昨天结束了

先说昨天的新闻：**Claude Code 2.1.277 开始支持 AGENTS.md**（9 月 18 日）。

The Register 的标题起得很直接——**"Anthropic 决定支持 OpenAI 的 markdown 指令规范"**。

如果你不是天天泡在 AI 编程工具里，可能会觉得：一个配置文件名，至于吗？

**至于。因为这场拉锯的起点，是 Shopify CEO 公开放的一句狠话。**

时间倒回 8 月 26 日。Shopify CEO **Tobi Lütke** 在 X 上表态：

> 我正考虑在 Shopify 禁用 Claude Code，**直到他们愿意读取 AGENTS.md、.agents/skills 等文件。**

一家电商巨头，威胁要全公司禁用一款主流 AI 编程工具——**原因只是它不肯读一个 markdown 文件。**

而他给出的理由，我觉得比新闻本身更有价值：

> AGENTS.md 和 CLAUDE.md 都会沿目录树递归生效。数千名开发者维护一个 monorepo，**难免有目录漏掉一份文件，结果就是一部分开发者用上"被切掉一块脑子"的智能体**。
>
> 我们用自动化解决了问题，但这是笔**愚蠢的复杂度税**，根本不该付。

**"被切掉一块脑子的智能体"**——这个说法，凡是维护过大型仓库的人应该都会心一沉。

## 先把 AGENTS.md 说清楚

AGENTS.md 的官方定义只有一句话：

> **Think of AGENTS.md as a README for agents.**

**给 agent 看的 README。**

README 给人看——怎么快速上手、项目是做什么的。而 AGENTS.md 装的是 agent 需要、但不适合塞进 README 的东西：**构建步骤、测试怎么跑、代码约定、几条铁律。**

它现在的地位比很多人想的高：

- 已被 **6 万+ 开源项目**采用
- 起源于多家协作：**OpenAI Codex、Amp、Google 的 Jules、Cursor、Factory**
- 现在由 **Linux 基金会旗下的 Agentic AI Foundation 托管**

官方给的最小示例长这样：

```markdown
# AGENTS.md
## Setup commands
- Install deps: `pnpm install`
- Start dev server: `pnpm dev`
- Run tests: `pnpm test`

## Code style
- TypeScript strict mode
- Single quotes, no semicolons
```

**就这么简单。** 没有 schema，没有特殊语法——就是一份 markdown。

而它真正好用的地方在于：**一份文件跨多个 agent 通用**。Codex、Cursor、Jules、Copilot、Claude Code 都读它，你不用为每个工具维护一份配置。

> 顺带一提，Claude Code 之前坚持用自己的 `CLAUDE.md`。开发者从 2025 年 8 月就开始提需求，相关 issue 攒了近 **5000 个 GitHub 点赞**，最后被关了，回复是"不予修复"。**直到 Shopify 威胁禁用，才等来了 9 月 18 日的支持。**

## 为什么"一份文件"会升级成平台工程问题

很多人会想：不就是让 Claude Code 多读一个文件吗，为什么 Shopify 要上升到"全公司禁用"？

**因为在一个几千人的 monorepo 里，一个文件格式差异会变成失控。**

Lütke 说的那个机制值得抄下来：

```
monorepo/
├── AGENTS.md          ← 全局规则（技术栈、提交规范）
├── backend/
│   └── AGENTS.md      ← 后端特有（不要动 db/migrations）
├── mobile/
│   └── AGENTS.md      ← 移动端特有（打包、签名）
└── infra/
    └── AGENTS.md      ← 基础设施特有
```

**AGENTS.md 是"就近生效"的**——agent 读到当前目录最近的那份，继承上层的规则。

问题就出在这儿：

- 假设团队同时用 Cursor（读 AGENTS.md）和 Claude Code（只读 CLAUDE.md）
- `mobile/` 目录下写了 `AGENTS.md`，但没人记得再写一份 `CLAUDE.md`
- 用 Cursor 的开发者：agent 知道"打包前要跑签名脚本"
- 用 Claude Code 的开发者：agent **不知道**

**同一份代码，同一个任务，两个 agent 给出的结果不一样。** 而这区别不会被任何报警发现——**你只是偶尔发现某个同事的 AI"特别笨"。**

**这就是 Lütke 说的"复杂度税"**：不是不能解决，而是**这个问题根本不该存在**。

## 落到实操：一个 KMP + AI Chat 客户端怎么写

好了，新闻讲完，接下来是正题。

假设我们在做的是一个 **Kotlin Multiplatform + Compose Multiplatform 的 AI Chat 客户端**——一套 Kotlin 代码，跑 Android 和 iOS。

这种项目写 AGENTS.md 有它的特殊性：**同一个仓库里，三种代码的规则完全不同。**

先看目录树：

```
ai-chat/
├── AGENTS.md                      # 根：全局铁律 + 通用命令
├── settings.gradle.kts
├── gradle/
│   └── libs.versions.toml         # 版本目录（依赖版本唯一来源）
├── shared/                        # 共享业务逻辑（无 UI）
│   ├── AGENTS.md                  # 这一层的规矩
│   ├── build.gradle.kts
│   └── src/
│       ├── commonMain/kotlin/     # ← 三端共享
│       ├── androidMain/kotlin/    # ← Android 实现
│       ├── iosMain/kotlin/        # ← iOS 实现
│       └── commonTest/kotlin/
├── composeApp/                    # Compose Multiplatform UI
│   ├── AGENTS.md
│   └── src/
│       ├── commonMain/
│       ├── androidMain/
│       └── iosMain/
├── androidApp/                    # Android 壳工程
│   ├── AGENTS.md
│   └── src/main/AndroidManifest.xml
└── iosApp/                        # iOS 壳工程（Xcode）
    ├── AGENTS.md
    └── iosApp.xcodeproj
```

**注意：这个结构本身就是给 agent 的第一层信息。** 如果 agent 分不清 `shared/` 和 `composeApp/` 的区别，它就会把 UI 代码写到逻辑层里。

### 根 AGENTS.md：只写"全局铁律"和"通用命令"

```markdown
# AGENTS.md

KMP + Compose Multiplatform 的 AI Chat 客户端。
一套 Kotlin 代码，编译到 Android / iOS。

## 项目结构
- `shared/`     — 共享业务逻辑，**不含任何 UI**
- `composeApp/` — Compose Multiplatform UI（三端共用同一套界面）
- `androidApp/` — Android 壳（Activity / Application / manifest）
- `iosApp/`     — iOS 壳（Xcode 工程 / SwiftUI 入口）

## 构建与测试
- 全量编译：      `./gradlew build`
- 共享层编译：    `./gradlew :shared:compileKotlinMetadata`
- Android 装到机：`./gradlew :androidApp:installDebug`
- 跑全部测试：    `./gradlew allTests`
- 只跑一个类：    `./gradlew :shared:allTests --tests "*ChatRepository*"`
- iOS 框架：      `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`

## 铁律（违反会导致三端编译失败）
1. `commonMain` 里**禁止**出现 `java.*`、`android.*`、`platform.*`
2. 平台差异一律走 `expect` / `actual`，不要写 `if (isAndroid)`
3. **不要为 iOS 复制一份逻辑** —— 那就失去了用 KMP 的意义
4. 新依赖先加到 `gradle/libs.versions.toml`，模块里只引别名，**不要写死版本号**
5. 改 `commonMain` 之后，必须能同时编译 Android 和 iOS
```

**这里有几个写法上的讲究，我认为很关键：**

**① 命令要"可复制执行"，不要描述。**
写 `./gradlew :shared:compileKotlinMetadata`，不要写"编译一下共享模块"。**agent 拿到能直接跑的命令，才不用猜。**

**② 铁律要写"违反的后果"。**
"禁止在 commonMain 用 java.\*" 是个规定；但加上"**会导致三端编译失败**"，agent 才知道这是硬约束，不是风格偏好。

**③ 用"唯一来源"消灭歧义。**
"版本号只写在 `libs.versions.toml`"——这一句能让 agent 不再到处硬编码版本。

### shared/AGENTS.md：只写"这一层特有的"

**子目录的 AGENTS.md 不该重复根目录的内容**，只写"进到这个目录才需要知道的事"：

```markdown
# shared/ — 共享业务逻辑

## 三端目录的职责
- `commonMain/` — 平台无关逻辑。要用平台能力，在这里先 `expect` 声明
- `androidMain/` — `expect` 的 Android 实现（用 Android SDK / JVM API）
- `iosMain/`     — `expect` 的 iOS 实现（用 Kotlin/Native + Foundation）
- **每加一个 `expect`，必须同时补齐 androidMain 和 iosMain 两份 `actual`**

## 目录约定
- `data/`      — Repository、数据源、DTO
- `domain/`    — 领域模型、UseCase
- `platform/`  — expect/actual 的平台抽象
                   （KeyValueStore、HttpEngine、AppInfo…）

## 网络（AI Chat 的核心）
- 统一用 **Ktor**，禁止在共享层直接用 OkHttp 或 URLSession
- 流式响应统一返回 `Flow<ChatChunk>`，不要在 shared 里做 UI 状态管理
- 超时、重试、错误映射统一在 `data/remote/` 里处理

## 命令
- 测试：      `./gradlew :shared:allTests`
- iOS 框架：  `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`
```

**看见没，这一层装的全是"KMP 特有"的知识**：

- `expect`/`actual` 的配对规则（这是 KMP 最容易出错的地方）
- 网络层为什么必须用 Ktor（因为 OkHttp 在 iOS 上不存在）
- 流式响应为什么用 `Flow`（AI Chat 的核心特征）

**这些是通用 README 里不会写、但 agent 每次都会踩的东西。**

### 三端壳工程：各自写"别乱碰什么"

**androidApp/AGENTS.md**：

```markdown
# androidApp/ — Android 壳

## 这里只放什么
- Activity、Application、AndroidManifest.xml、res/ 资源
- **业务逻辑一律放 shared/，UI 一律放 composeApp/**

## 常用命令
- 装机：    `./gradlew :androidApp:installDebug`
- 看日志：  `adb logcat -s AiChat:V`

## 注意
- 新增权限必须改 AndroidManifest.xml，并**同步更新本文件**
- 改 minSdk 前先确认 shared/ 里用到的 API 兼容性
```

**iosApp/AGENTS.md**：

```markdown
# iosApp/ — iOS 壳（Xcode 工程）

## 重要：这里不用 Gradle 构建
- 这是 Xcode 工程，**不要尝试用 gradlew 构建它**
- 改了 shared/ 的代码后，先跑：
    `./gradlew :shared:linkDebugFrameworkIosSimulatorArm64`
  让 Xcode 拿到新框架，再去 Xcode 里 Run

## Swift 侧只做桥接
- 生命周期、权限申请、导航容器
- 业务逻辑**全部回 shared**，不要在 Swift 里重写一遍

## 常见坑
- Info.plist 缺权限描述：模拟器运行才报错，Xcode 不提示
- 改了 framework 的 export 后要 Clean Build Folder
- Kotlin 的 sealed class 在 Swift 侧要遍历所有分支（写全，别用 default）
```

**为什么这三份要分开写？**

因为**"该做什么"和"不该做什么"在不同目录里完全相反**：

- 在 `shared/`，你要写"不要碰平台 API"
- 在 `androidApp/`，你要写"不要写业务逻辑"
- 在 `iosApp/`，你要写"不要用 Gradle 构建这个目录"

**同一个仓库，规则互相冲突。** 一份根 AGENTS.md 写不下这些——**这就是"就近生效"存在的意义。**

### 顺带一提：如果团队里工具是混用的

AGENTS.md 只是"约定"，不同工具读它的方式仍有差异。官方给了一些配置参考：

- **Aider**：在 `.aider.conf.yml` 里加一行 `read: AGENTS.md`
- **Gemini CLI**：在 `.gemini/settings.json` 里配置 `{ "context": { "fileName": "AGENTS.md" } }`
- **老文件迁移**：`mv AGENT.md AGENTS.md && ln -s AGENTS.md AGENT.md`（软链保留兼容）

放到 KMP 项目里，还有一种情况是**工具版本不齐**——比如有人的 Claude Code 还没升到 2.1.277。这时候可以在关键目录放一个 `CLAUDE.md -> AGENTS.md` 的软链顶上。

**但请记住 Lütke 那句话：这就是在付"复杂度税"。**

**能统一升级工具，就别维护两份文件**——软链是过渡方案，不是终局。

## 几个我踩过的坑

**① 别把 AGENTS.md 写成 README 的翻译版。**
README 是给人看的"项目介绍"，AGENTS.md 是给 agent 的"操作手册"。区别在于：**README 讲 what，AGENTS.md 讲 how，而且要能直接执行。**

**② 别写"应该是""建议""尽量"。**
agent 对模糊表述的处理方式是不可预测的。要么是硬约束（写清后果），要么就不要写。

**③ 别让它过期。**
**过期的命令比没有命令更糟**——agent 会照着跑，然后卡在一个已经不存在的任务上。改构建脚本时顺手改 AGENTS.md，应该成为习惯。

**④ 别贪多。**
官方那个最小示例就十几行，够用了。上下文是有预算的，**你写给 agent 的每一行，都在挤占它理解代码的空间。**

**⑤ 目录树画出来。**
这个投入产出比最高。**一张准确的目录树，能省掉 agent 十次 `ls`。**

## 我的看法

**第一，AGENTS.md 的胜负，本质是"平台中立"对"工具绑定"的胜利。**

Anthropic 拖了近一年才支持，最后还是支持了。原因很简单：**当 6 万个项目用同一个格式，你不支持，就是在给用户加税。**

Shopify 那句"复杂度税"，戳中的是所有大仓库的真实痛点。

**第二，对 KMP 项目来说，AGENTS.md 的价值可能比对普通项目更高。**

因为 KMP 的坑**特别"目录相关"**：

- 这段代码在 `commonMain` 还是 `iosMain`？规则完全不同
- 这个 API 三端都有吗？
- 改完要不要重新 link iOS 框架？

**这些判断，如果每次都靠人在对话里说一遍，成本很高；写进目录级的 AGENTS.md，就变成了一次性投入。**

**第三，这件事的长期意义可能比看起来大。**

当 AI 编程工具从"补全代码"走向"独立完成任务"，**"仓库里有没有一份说清楚规矩的文件"**，会越来越像"这个项目有没有自动化测试"——**是工程成熟度的标志之一。**

而且写它还有个附带好处：**写 AGENTS.md 的过程，本身就是一次团队规范的梳理。** 你会被迫把那些"只有老员工才知道"的隐含规则写下来——**这件事的价值，可能比让 AI 聪明一点更大。**

---

想问问你：

**你现在的项目里有 AGENTS.md 吗？** 如果还没有，你觉得最难写的是哪部分——是构建命令，还是那些"只可意会"的约定？

**或者你在 KMP 项目里踩过什么坑，后来靠文档解决了？** 评论区聊聊，我可以整理成下一篇。

---

*参考资料：AGENTS.md 官方规范（agents.md）、The Register（9/18）、36Kr / CSDN 关于 Shopify 与 Anthropic 争议的报道（8/26）、Claude Code 2.1.277 更新说明。*
