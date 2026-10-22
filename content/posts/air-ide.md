---
title: "agent 一多，IDE 开始抢调度权"
date: 2026-10-02T19:29:41+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "我本机终端里现在装着三个 agent：claude、opencode、gemini。用几天就撞上一个新麻烦，会话一多，我根本不知道谁在跑、谁改了哪些文件、谁花了多少钱，全靠翻终端历史去认。"
cover:
    image: "/images/air-ide/cover.png"
    alt: "agent 一多，IDE 开始抢调度权"
    relative: false
---

## 先说一件小事

我本机终端里现在装着三个 agent：claude、opencode、gemini。用几天就撞上一个新麻烦，会话一多，我根本不知道谁在跑、谁改了哪些文件、谁花了多少钱，全靠翻终端历史去认。

JetBrains 显然也在想这件事，而且给的答案比"再做一个聊天框"大得多。

## 它的答案叫 Air

9 月 22 日，JetBrains 的 CEO Kirill Skrygan 亲自写了一篇博文，把过去半年散着做的东西收拢成一个体系，名字叫 Air。

里面有三个产品：Air in JetBrains IDEs（在 IDE 里指挥 agent）、Air Teams（把个人的 agent 活动变成团队工作流）、Air Governance（组织级的可见性与成本治理）。

开头那句话我觉得写得挺实在，原文贴一下：

> AI can produce code. Organizations still have to produce software. Agentic development is changing how software gets made, but it hasn't changed what it costs to be wrong.

里面还有一句，是我认为整篇最值钱的：

> Code becomes cheaper to generate but more expensive to verify.

翻译过来就是：写代码变便宜了，验证变贵了。所以他们把赌注押在"验证"这一侧。

10 月 1 日，Air in IDEs 开了 EAP。

## 它的定位：不是 AI 提供方，是集线器

官方描述里有一句很关键：

> It is not an AI provider, and it ships with no agents installed.

翻译一下：它不做模型，一个 agent 也不预装，它做的是把你的 agent 接进来。三个设计要点复述一下：多个 agent 并行工作、每个会话用临时 worktree 隔开、你能看到每个会话的开销。另外，双击 Ctrl 就能起一个新会话，当前上下文自动带上。

JetBrains 在这件事上姿态放得很低，原话是"我们建了自己的 agent，也打算把它做到最好，但 Air 不要求你用我们的"，还说"我们没理由把生态做小"。它敢这么说，是因为它手里有别的东西：26 年的代码智能。

## 我把它装上了

先讲一个好用的结论：不用装 EAP 也能试。插件市场上有两条兼容线，一条对应 2026.3 EAP，一条对应 2026.2 稳定版，我选了后者（版本 262.8665.481，当时累计下载 33,519 次）。

我装在 Android Studio 2026.2（build 262.9437）上。插件要求的平台版本是 262.8665，装上去兼容。启动之后日志里这一行是第一个证据：

```text
Loaded custom plugins: ... Air (262.8665.481), Kotlin Multiplatform (262.9437.115-AS), ...
```

有意思的是后面这几行，我觉得比插件本身更值得看：

```text
INFO - AcpRegistryFetcher - Starting ACP registry fetcher
INFO - AcpRegistryFetcher - Fetching signed registry bundle from
       https://resources.jetbrains.com/agentclientprotocol/registry.zip
INFO - AcpRegistryFetcher - Verified signed registry v1.0.0 with 41 agents
INFO - AcpAgentsConfigService - ACP configuration file
       '/Users/trevorwang/.jetbrains/acp.json' not found,
       no local ACP agents will be configured.
INFO - CodexIdeContextIpcTransport - Codex IDE context IPC listening on
       /var/folders/.../codex-ipc/ipc-501.sock
INFO - AirCloudRunAvailabilityServiceImpl - Air cloud runs: not offered
       - no JetBrains Account is signed in
```

![本机 idea.log 的真实日志，Air 启动后拉取签名目录的那几行](/images/air-ide/idea-log.png)

这六行分别说明：它启动就去拉一个签名的 agent 目录；目录里有 41 个 agent；本地自定义 agent 的入口是 `~/.jetbrains/acp.json`；它会给 Codex 开一个本机 IPC 插座，让 agent 能问 IDE 要上下文；云端跑需要登录 JetBrains 账号，没登录就不提供。

还有一条我没想到的：Android Studio 2026.2 的自带插件列表里就有 `Agent Client Protocol`。也就是说 ACP 支持不是只从这个新插件来，IDE 本体已经带了。

## 那 41 个 agent 都是谁

那个目录是签名分发的，我把它拉下来自己解了：里面是 `registry.json`、`registry.json.asc` 和一个公钥环。我自己验了一遍签名，结果是通过，公钥指纹 `B46DC71E03FEEB7F89D1F2491F7A8F87B9D8F501`。

41 个里面，挑你眼熟的：

- Claude Agent、Codex、Gemini CLI、GitHub Copilot、Cursor
- Devin、Amp、Cline、goose、Junie、Factory Droid、OpenCode
- Kimi CLI、Qwen Code、MiniMax Code、GLM Agent、Mistral Vibe、Qoder CLI

国内的几家都在里面。

装法最能说明这个目录的性质，我贴四个真实条目：

```text
Claude Agent 0.85.1   npx @agentclientprotocol/claude-agent-acp@0.85.1  (license: proprietary)
Codex 2.1.1           npx @agentclientprotocol/codex-acp@2.1.1           (Apache-2.0)
Gemini CLI 0.62.0     npx @google/gemini-cli@0.62.0 --acp                (Apache-2.0)
OpenCode 1.18.34      直接给各平台二进制 + sha256 校验值                  (MIT)
```

所以"装一个 agent"在这套体系里就是：一条 npx 命令，或者一个带校验值的二进制包。注意 Claude 那一行，包装是 proprietary，这个细节做企业合规的人得看一眼。

## 我自己跟 ACP agent 对了一次话

GUI 我没点成，本机没给这个进程截图和辅助功能权限，鼠标点不了。所以我换了个更硬的验证办法：绕开 IDE，直接在终端把注册表里那个 Claude 的 ACP 包装跑起来，手写 JSON-RPC 跟它握手。

这是真实响应，不是我编的：

```json
{"protocolVersion": 1,
 "agentCapabilities": {
   "_meta": {"claudeCode": {"promptQueueing": true}},
   "promptCapabilities": {"image": true, "embeddedContext": true},
   "mcpCapabilities": {"http": true, "sse": true},
   "loadSession": true,
   "sessionCapabilities": {"close": {}, "delete": {}, "fork": {}, "list": {}, "resume": {}}}}
```

它还主动推来一条通知，说自己没登录：

```text
_auth/status_update {'authStatus': {'kind': 'none', 'label': 'Not logged in'}}
```

发 `session/new` 之后，它返回了会话 ID 和四种模式：Manual（改之前先问）、Accept edits（自动接受改动）、Plan（先出计划）、auto。

最后我犯了个错，也值得写下来：发 `session/prompt` 时我图省事填了个假会话 ID，它回的是 `-32603 Internal error: Session not found`。也就是说这个协议必须带 `session/new` 返回的那个真实 ID，填错只给你一句内部错误，对新手不太友好。

三条结论：协议这端在我本机是活的；agent 会主动上报自己的能力与登录状态；认证没走完，所以我没拿到模型回答。真跑一轮对话，得先把授权做完，就是 IDE 里那一步。

## 为什么做 KMP 和 Android 的人值得看

第一，Android Studio 和 IDEA 同源，我上面那些日志就是 Android Studio 打出来的，ACP 插件还是自带的。JetBrains 是 KMP 的娘家，它往哪儿走，你的工具链就往哪儿走。

第二，它承认了一件事：

> The era in which the whole software development system can be contained in one window is ending.

一个窗口装不下整个开发流程了。这话从一个做了 26 年 IDE 的公司嘴里说出来，分量不一样。

第三，它的赌注明确押在多厂商上，理由也说得直白：模型各有所长、排行榜几个月一变，团队内部的选择本来就不一样。所以它不去赌谁赢，它要的是谁都得从它这儿过。

## 我的判断

第一，这次真正的动作是"调度权"。谁握着你那堆 agent 的调度台（谁在跑、动了什么、花了多少），谁就握住了开发者每天的入口。终端里当然也能编排，tmux 加脚本就行，但 IDE 手里有确定的代码智能，它说给 agent 用这套智能还能省钱，因为 agent 不用重新去发现代码里已有的信息。

第二，开放是策略，不是善意。ACP 是它和 Zed 一起推的开放协议，41 个 agent 里大部分不是它家的，连 Claude、Codex、Cursor 都在目录里。它不需要赌哪家赢，它要的是都从它这儿过一道。

第三，边界我照例标清楚。我核到的是：安装与版本兼容、插件确实加载、签名目录与 41 个条目、独立验签、ACP 协议握手与 agent 上报的能力。我没做成的是 IDE 界面里的多会话并行、worktree 隔离和成本显示这三样，本机权限不够，点不了。所以这一篇算"日志与协议层证据"，不算完整功能实测。再加上它还在 EAP，别急着换工作流。

参考资料：

- Air in IDEs EAP 博文：https://blog.jetbrains.com/ai/2026/10/air-in-ides-eap/
- 9 月 22 日 Air 发布博文（CEO 署名）：https://blog.jetbrains.com/blog/2026/09/22/introducing-jetbrains-air/
- ACP 协议官网：https://www.jetbrains.com/acp/
- ACP 注册表与提交入口：https://github.com/agentclientprotocol/registry
- Air 插件页：https://plugins.jetbrains.com/plugin/33314-air-alpha
- Android Studio 下载：https://developer.android.com/studio

验证边界：插件版本线与下载量取自 JetBrains 插件市场接口；Android Studio 版本号取自本机安装的 product-info.json；插件加载、注册表拉取、41 个 agent、acp.json 路径、Codex IPC 插座、云端需登录这几条，均取自本机 idea.log 的实跑日志。

41 个 agent 名单、各条目分发方式与 License 取自拉取到的 registry.json；签名由我本地用 pgpy 独立验签通过。协议握手的响应是终端实跑输出，认证未完成，因此没有取得模型回答。
