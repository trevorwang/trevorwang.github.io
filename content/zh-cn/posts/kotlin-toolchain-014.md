---
title: "Kotlin Toolchain 0.14 实测：agent 白名单"
date: 2026-11-19T07:00:00+08:00
draft: false
categories: ["公众号"]
tags: ["Kotlin", "AI Agent"]
summary: "0.14 加了 iOS 发版和 agent 配置。实测三平台全通、模拟器跑起来；但七个 agent 名字里没有 dsh，选 other 就拿不到 .mcp.json。"
cover:
    image: "/images/kotlin-toolchain-014/cover.png"
    alt: "Kotlin Toolchain 0.14 实测：agent 白名单"
    relative: false
---

六周前我写过一篇，说 Amper 转正成了 Kotlin Toolchain，结尾那句是「iOS 那条腿停在了 Xcode 上」。

11 月 17 日 0.14 发了，头号新功能就是 iOS 应用发布，一条命令上 TestFlight。我升上去试了，从建工程到三平台全通、模拟器跑起来，再到发版向导，一路走到底。

## 0.14 改了什么

官方博客列了六条。

最重的是 iOS 应用发布，`kotlin publish testflight`，交互式向导带着走完首次发布，之后同一条命令发更新。要 App Store Connect 的 API key，官方强调代码里从不保存私钥，也不要 Apple ID 密码。

第二条是 agent 那侧。CLI 向导现在会生成 MCP server 配置和 `AGENTS.md`，同期还发了 Kotlin Toolchain 的 agent skills，装法是 `npx skills add kotlin/kotlin-agent-skills`。

官方说他们自己的 AI evals 显示，用上这套 skills 能减少 token 消耗。

剩下四条小一些。IDE 里模板引起的错误会同时标在源头和模板应用两处；Android 支持 core library desugaring；外部工具的下载地址全部文档化并且能用环境变量覆盖；依赖解析优化，官方 benchmark 说无缓存首次导入快了三倍以上。

![0.14 官方列的六条更新](/images/kotlin-toolchain-014/fig1.png)

## 升级本身是三十秒的事

本机原来装的是 0.13.0，落在 `~/.local/bin/kotlin`。

我重跑了一遍官方安装脚本 `curl -fsSL https://kotl.in/install.sh | sh`，它下了新的 CLI 运行时 v25.38.109，装完版本变成 0.14.0，commit 95a1580，构建日期 2026 年 11 月 16 日。

没有残留，没有要手动清理的旧目录。这一条比大多数工具做得干净。

## 建工程时，输出里有一段是给 agent 的

我用 `kotlin new` 建了个三平台工程，安卓、iOS、桌面，指定 `--agent=other`。

跑完 CLI 在收尾处打了一段话，原文是这样的：这个项目配置了 MCP server，比如 Compose Hot Reload，但它们在当前会话里不可用，因为 MCP server 只在会话启动时加载，请让用户从项目目录重启你，这样你就能用它们了。

这段话不是给我的。它是在让 agent 转告我一句话。

![七个 agent 名字，选 other 就少一个文件](/images/kotlin-toolchain-014/fig2.png)

再看 `kotlin new --help` 的末尾，还有一段：创建项目时总是要传 `--agent=<你的 agent 名>`，这有助于设置 AGENTS.md、skills、MCP server 和其他 AI 功能，只有在用户明确要求时才传 `--no-ai`。

help 文档里写「总是要传」，主语是 agent。这东西是照着一个 agent 会读 help 来设计的。

## AGENTS.md 里有一条「不要写 sleep 循环」

生成的 `AGENTS.md` 只有 23 行，但密度很高。

它先告诉你用项目自带的 `./kotlin` wrapper，不要用全局命令。然后讲 UI 开发的正确姿势：改代码之前先把 app 起起来，带 `--compose-hot-reload` 参数放后台跑，紧接着调 MCP 的 `status`，反复调直到返回 `connected: true`，因为首次构建可能要几分钟。

后面这句是我最想抄的：`status` 是唯一的就绪信号，不要 tail 或者 grep 运行日志，不要写 sleep 循环。

这条规则一看就是从真实失败里长出来的。我见过太多 agent 卡在这种地方，写个 `sleep 30` 然后去读日志，读到一半误判成功。

配套的还有：启动失败别猜，看 `status` 里的 `lastError` 和 MCP 的 `get_logs` 工具；连上以后改完 UI 用 `reload` 应用，如果开了 `buildContinuous` 就用 `await_reload`。

## 但 agent 得在名单里

`--agent` 支持七个值：claude、codex、antigravity、junie、pi、opencode、other。

我先用 `--agent=other` 建了一个，又用 `--agent=claude` 建了一个，同一个命令，同一个项目参数，对比生成的文件。

claude 那个多一个 `.mcp.json`，other 那个没有。

两个版本的 `AGENTS.md` 逐行比对，完全一致。差别就在这一个文件上。

而 CLI 的输出在两版里都说了「this project configures MCP servers」。对 other 来说这句话只兑现了一半：它给了你一份说明，让你自己去配。

## skills 装完，我的 agent 被跳过了

官方给的 skills 安装命令是 `npx skills add kotlin/kotlin-agent-skills`。我在项目目录里跑了。

装到 10 个 skill，落在 `.agents/skills/` 下面，`.claude/skills` 那边也生成了一份。

有意思的是它的安装日志。每个 skill 后面跟着三行归类：universal 一档列了 Antigravity、Gemini CLI、OpenCode、Pi、Warp 加 16 个，symlinked 一档只有 Claude Code。

skipped 那一档写着 OpenClaw、Command Code、Hermes Agent、Junie，理由都是 project directory not found。

Hermes Agent 就是我现在用的这个。它被认出来了，但被跳过了。

这十个 skill 里，只有两个在自己头上写了测试信息。`kotlin-tooling-kotlin-toolchain` 写的是 `anthropic/claude-sonnet-5, anthropic/claude-opus-5`，agent_version 是 claude-code，last_eval 是 2026 年 8 月 14 日。

另一个 `kotlin-tooling-native-build-performance` 写的是 `openai/gpt-5.5, openai/gpt-5.4-mini`，last_eval 是 7 月 6 日。

其余八个都没写。

官方那句「我们的 AI evals 显示用上 skills 能减少 token 消耗」没错，但落到具体文件上，只有五分之一带了评测记录，测的还是 Claude 和 GPT 两条线。

## 那个 .mcp.json 里有两个 server

claude 版的配置文件是这样。

一个叫 `compose-hot-reload`，命令是 `./kotlin compose-hot-reload-mcp-server`，走本地。

另一个叫 `klibs`，类型是 http，地址 `https://api.klibs.io/mcp`。

第二个官方博客一个字都没提。我去翻 0.14 的更新说明和文档，讲 agent 那节只讲了 AGENTS.md 和 skills，没讲这个 klibs 服务。

它大概率是给 agent 查 Kotlin 库信息用的，接在 klibs.io 那个库索引站后面。

![.mcp.json 里的两个 server](/images/kotlin-toolchain-014/fig3.png)但这个发现只能从生成的配置文件里挖出来，文档里找不到。

## 四次构建，实测数据

工程带三个平台，build 默认全平台。我按平台分别跑了一遍，拿到这组数。

`./kotlin build -p jvm` 10.0 秒，增量，最快。

`./kotlin build -m desktopApp` 53.2 秒，中间自动下了 JDK 25.0.5。

`./kotlin build -m androidApp` 78.8 秒，中途自动下了 Gradle 9.8.0 和 Android commandlinetools。

`./kotlin build` 不带参数，2 分 45 秒，全平台一次过。

前三次是有意分开跑的，因为我想知道工具链到底把哪些东西管进了自己的怀里。答案是全管：JDK、Gradle、Android SDK、Kotlin/Native 预编译包、xcbeautify，一个不用你提前装。

这也是 0.14 第五条改动的实证，外部工具的下载地址全部文档化，还能用环境变量覆盖走内网镜像。

![四次构建的耗时对照](/images/kotlin-toolchain-014/fig4.png)

desugaring 那条，博客的原话是在 `module.yaml` 里加一行 `desugaring: enabled`。我照字面加到顶层，直接报 `Unknown property 'desugaring'`，连行号列号都给了。

挪到 `settings` 下面的 `android` 里才通过。

这条博客写得省了，正文那句会让人加到顶层。它后面给了文档链接，但正文不该这么写。

![desugaring 的正确位置](/images/kotlin-toolchain-014/fig5.png)

## 全平台构建：2 分 45 秒，两端都出活

`./kotlin build` 不带参数，2 分 45 秒，Build successful。

日志逐条过：shared 的 android tests、iosArm64、iosSimulatorArm64，加上后两者的测试编译，全部成功。iosApp 的 iosArm64 和 iosSimulatorArm64 也都成功。

中途自动下了 kotlin-native-prebuilt 2.4.20 和 xcbeautify 3.2.1。

有一条警告值得记：DEVELOPMENT_TEAM 这个 build setting 在 Xcode project 里没检测到，工具链自己把 CODE_SIGNING_ALLOWED=NO 加了进去，禁用签名，留你事后手动签。这个处理算体面，没闷声烧工程。

再往下跑，`./kotlin run -m iosApp`，它把 app 装进了 iPhone 17 模拟器。

紫色的 Compose 默认界面，中间一个 Click me! 按钮。进程 pid 40232，launchctl 里挂着 UIKitApplication:dev.lazy.ktc014。

六周前那句「iOS 那条腿停在了 Xcode 上」，这回真接上了。

![iOS 模拟器里跑起来的界面](/images/kotlin-toolchain-014/ios-sim.png)

## TestFlight：现在只差一张 key

`kotlin publish testflight -m iosApp`，它直接进了「Connect your Apple Developer account」向导。

一步一句。去 App Store Connect 的 Users and Access 找 team API key，自动 provisioning 要用 Admin 级 key，前提是 Apple Developer Program 会员加 API 访问权限。

下载 .p8 私钥存到安全的地方，别进版本库。然后它要私钥路径、Key ID、Issuer ID 三项。

它还补一句「Kotlin Toolchain 从不复制也不保存这个文件」。填的值存进模块目录的 `app-store-connect-auth.properties`，`.gitignore` 自动加一条。

我手里没有 .p8，卡在收尾那句：A readable private key path, Key ID, and Issuer ID are all required。然后它很客气地让我准备好了再来。

所以这条链路只剩一道门槛，就是 Apple 开发者账号。工具链这侧该做的都做了。

从 help 里能读到的边界也一并说清。

三个环境变量 `KOTLIN_TOOLCHAIN_ASC_KEY_PATH`、`KOTLIN_TOOLCHAIN_ASC_KEY_ID`、`KOTLIN_TOOLCHAIN_ASC_KEY_ISSUER_ID` 是 CI 走的路径，文件优先于环境变量。

一次只能发一个 iOS app，多 app 工程要 `-m` 指定。签名只支持 straightforward setup 的自动模式，复杂场景它直接让你上 fastlane。

还有一条没说在博客里，但只要用上就绕不开。

MCP server 是靠 `build/hot-reload-app.pid` 找 app 的，那文件里写着 app 的 pid 和一个 `orchestration.port`。app 不在跑，那个文件就是空的或者过期，`status` 永远连不上。这是 AGENTS.md 要求先起 app 的原因。

## 我的判断

0.14 这版两头都有动作。

agent 那侧是真往前走了一步：AGENTS.md 里的规则写得务实，MCP server 能应答，pid 文件的机制也扎实。问题出在白名单上，七个 agent 里没有 dsh，我只能选 other，于是拿不到 `.mcp.json`。CLI 说「这个项目配了 MCP server」，然后只给我一份说明书让我自己照着配。

iOS 那侧走通了大半。三平台全平台构建一次过，模拟器装得上也跑得起来，发版链路接上 App Store Connect 也通了，只差一张开发者账号的 .p8。这条线在 0.13 时代断在半路，现在从建工程到出产物是一条直线，中间不用你手配 Xcode project、不用写 Gradle 脚本、不用知道 kotlin-native 装在哪。

所以这次升级值得做。前提是你的 agent 在名单里，那部分多出来的东西是实的；不在名单里你依然能用，只是得自己补那半个文件。

iOS 那边，你缺什么它会直接告诉你，缺账号就只缺账号。

你用的 agent 在这个名单里吗？
