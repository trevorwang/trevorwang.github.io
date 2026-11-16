---
title: "Air 的真身：JetBrains 把 IDE 拆成 23 个工具交给 agent"
date: 2026-11-16T08:04:36+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "11 月 11 日，JetBrains 发了篇《JetBrains IDEs Transformed for the Agentic Era of Development – With Air》，作者 Dominique Rolink。Air 随 2026.3 版本进 JetBrains IDE。"
cover:
    image: "/images/air-ga/cover.png"
    alt: "Air 的真身：JetBrains 把 IDE 拆成 23 个工具交给 agent"
    relative: false
---

11 月 11 日，JetBrains 发了篇《JetBrains IDEs Transformed for the Agentic Era of Development – With Air》，作者 Dominique Rolink。Air 随 2026.3 版本进 JetBrains IDE。

开篇就摆了个数据：49% 的开发者每周花四小时以上审阅 AI 产出、纠正 prompt，出处是 Atlassian 的 State of Developer Experience Report 2026。他们的结论是，agent 写得越多，个人产出越高，瓶颈就从写代码移到了审阅。

官方对 Air 的定义是：把编码 agent 带进你现有的开发环境，一个地方跟踪进度、审阅改动、回送反馈。核心卖点那句是「因为跑在 IDE 里，它把 IDE 的代码智能带给了你和你的 agent」。

这句话才是重点。我把本机那两个 IDEA 版本翻了一遍，想看看「把代码智能带给 agent」具体是怎么做到的。

## 先说一件容易被忽略的事

Air 插件的自述里有一句：`Air connects your IDE to your agents. It ships with no agents installed.`

翻译过来是，Air 只负责把 IDE 连到 agent，它本身一个 agent 都不带。

这句话决定了它的定位。它不是又一个 AI 助手，是一个接线板。

![Air 的定位：接线板，不带 agent](/images/air-ga/fig5.png)Codex、Claude Agent、GitHub Copilot、Junie 这些是插座上的电器，通过 ACP 协议插进来，你付的还是原来那家的钱。官方明说用第三方 agent 不需要 JetBrains AI 订阅。

那么问题来了：一个接线板凭什么值一个 GA 发布。

## 答案在插件目录里

我在本机装了两个版本的 IDEA。一个是日常在用的 2026.2.3，另一个是十月底从 DMG 直接跑过的 2026.3 EAP，现在还挂在 `/Volumes` 上。

对比这两个版本的 `Contents/plugins`，差别很清楚。

2026.2.3 里有 `mcpserver` 和 `debuggerMcp` 两个插件。2026.3 里变成了五个：`mcpserver-plugin`、`debuggerMcp`、`java-coverageMcp`、`java-debugger-agent-bridge`，加上新的 `air-plugin`。

也就是说，MCP 基建在 2026.2.3 就已经在 IDE 里了，Air 是后加的那层调度台。

这个顺序很关键。JetBrains 不是先做了个聊天窗口再想办法接工具，而是先把 IDE 拆成 MCP 工具集，再补上让 agent 入住的前台。

![两个版本的插件目录对比](/images/air-ga/fig1.png)

## 2026.2.3 的 MCP Server 已经在做什么

`mcpserver.jar` 里的 plugin.xml 写着它的能力清单：读取当前文件、执行终端命令、设置断点、搜索项目文件，后面跟一句 and more。

依赖里有 `ktor-server-sse-jvm.jar` 和 `io.github.smiley4.schema.kenerator`，说明当时用的是 SSE 传输，schema 生成接的是第三方库。

到 2026.3，这层依赖换了一遍：`ktor-server-sse.jar` 换成了平台统一的 `intellij.libraries.ktor.server.sse.jar`，schema 生成统到 `intellij.libraries.schema.kenerator.jar`，还多了一个 4.7MB 的 `intellij.libraries.mcp.kotlin.sdk.jar`，也就是官方的 MCP Kotlin SDK。

真正的功能变化在这个文件：`intellij.mcpserver.stdio.jar`。2026.2.3 里没有它。

SSE 是给远程客户端连的，stdio 是给本地进程连的标准方式。Codex CLI、Claude Code 这类跑在终端里的 agent，走的就是 stdio。Air 能让这些终端 agent 用上 IDE 的能力，靠的就是这一块。

![MCP 传输从 SSE 到加 stdio](/images/air-ga/fig4.png)

## 四个 skill，教 agent 怎么用 IDE

插件之外还有一层东西，藏在缓存目录的 `ai-agent-extensions/agents/.agents/skills` 下。本机这个 2026.3 EAP 里启用了四个：

- `ij-debugger`，来自 IDEA，owner 写着 IJPL Profilers & Debuggers
- `command-monitor`，来自 Codex 监控
- `database-connection-management`，来自 DataGrip
- `database-text-to-sql`，也来自 DataGrip

每个都是标准的 agent skill 结构：一个 `SKILL.md` 加 `metadata.json`，metadata 里记着版本和 sha256。文件名是 `.intellij-bundled-skill` 标记它由 IDE 自带。

这就是官方说的 built-in skills。agent 连上来以后不是瞎摸，IDE 直接塞给它一本说明书。

![四个 bundled skill](/images/air-ga/fig2.png)

## ij-debugger 是一本 32KB 手册

四个里面最重的是 `ij-debugger`。主文件 `SKILL.md` 有 32,426 字节，另外带 23 个工具的独立文档，每个 2 到 8KB。

工具分三组。JVM 那组五个：`jvm_debug_set_method_breakpoint`、`jvm_debug_set_exception_breakpoint`、`jvm_debug_list_method_breakpoints`、`jvm_debug_list_exception_breakpoints`、`jvm_debug_get_thread_dump`。运行配置那组两个：`get_run_configurations`、`execute_run_configuration`。

剩下一大组是 `xdebug_*`，十六个。设置断点、附加到进程、控制执行、求值表达式、取栈帧值、改改变量值、运行到指定行、取线程列表、取进程输出。

`xdebug` 是 PHP 的调试协议。也就是说，这本挂在 IDEA 名下的调试手册，一半以上篇幅在教 agent 调 PHP。合理的解释是这个 skill 同时服务 IDEA 和 PhpStorm，只是名字带了个 ij。

`debuggerMcp` 插件的自述可以解释这些工具从哪来：它基于 IntelliJ Platform 的调试 API 提供「完整功能的调试体验」，支持断点、追踪点、单步、栈检查、取值、表达式求值。

这说明工具不是套壳。agent 按下的断点，就是你在 IDE 里按的那个断点，走的是同一套平台 API。

![23 个调试工具的分组](/images/air-ga/fig3.png)

## 手册里最实在的一段是讲引号的

`SKILL.md` 开头叫 Invocation Mode，规定得死死的：先看 `execute_tool` 在不在，在就只用它，不在才直接调底层 handle，一个任务内不许混用。

然后是参数规则，逐条列出来：每个参数必须带值，裸 flag 不支持；布尔值要显式写 true 或 false。

最有意思的是这一条：含空格或操作符的值必须用引号包起来，表达式几乎总是这样。原因是路由器按空白切分命令，没加引号的多词值会被当成额外参数，报错长这样：

```
Invalid argument format: '+'. Expected '--paramName value' format
```

手册还专门给了两个例子，`--expression 'user.getName().trim()'`，以及表达式本身含单引号时外层改双引号，并且特意注明「不要对顶层表达式参数做 JSON 编码」。

这不是写给新人看的文档，是写给会踩坑的模型看的。JetBrains 显然是拿真实失败案例反推了这套规则。

## command-monitor 只服务一个 agent

另一份手册更挑对象。`command-monitor` 的第一句是「Run long commands, tests, builds, and watchers in Codex」，只提 Codex。

它规定只在 Full access 打开时用，判断标准是当前权限说明里的 `sandbox_mode` 是不是 `danger-full-access`，`workspace-write` 和 `read-only` 都算关。还不许为了用这个监控去申请 Full access。

用法也写得很细：`monitor_start(command, cwd)` 启动，不给时限的话默认无限制，要限就给 `deadline_seconds`，范围 1 到 3600。起服务器或者 watcher 要把它本身当命令传，不要加 `&` 或 `nohup`，因为监控会在命令退出时把后台子进程一起收掉。

等待策略是递进的：第一次 `timeout_seconds=30`，每次超时翻倍，30、60、120，之后固定 240。每个结果最多带 16KB 新输出。

这份手册甚至教 agent 怎么写 JS 调工具，连 `// @exec: {"yield_time_ms": 250000}` 这种注解都给了，还说明了不加这行会导致单元格 30 秒就返回。

## 免费的部分和消失的部分

博客里占了一小节的免费策略是：Junie 的 Lite 配置免费，登录 JetBrains 账号就能用，不要 AI 订阅。代码补全照常工作，而且不占 JetBrains AI 配额。

另一边，AI Assistant 还在，作为独立插件支持到 2027 年初。如果你升级后 Air 把 AI Assistant 顶掉了，连按两次 Shift 搜 Open AI Assistant 能找回来。

Air 自己也可以关，Settings 里的 Plugins 直接禁用重启。这个选项写得挺克制，官方那句是「作为坐在驾驶位的人，你决定用不用、用多少」。

## 我的判断

这次 GA 真正值钱的东西不在界面上。

界面部分，Air 能做的是并行 session、worktree 隔离、通知、preset、Code Notes 批注。这些功能别的工具多少都有。

值钱的是那套 MCP 工具集和 skills。2026.2.3 就已经把 IDE 拆成了可调用的工具，2026.3 补上 stdio 让终端 agent 也能接，再补上 Air 这个前台。三步走完，一个跑在 Claude Code 里的 agent 和你手动开 IDE 按断点，操作的是同一个调试器、同一套重构、同一个数据库连接。

顺手一个细节，博客里自己承认 worktree 有个缺口：agent 自己建的 worktree 目前在 Air 里还跟踪不到，官方说在做。

如果你本来就是 IDEA 深度用户，这套东西的意义是把 agent 从「另一个窗口」变成了「IDE 里的一个会话」。如果你在别的地方跑 agent，值得看一眼它的 MCP server，把 IDE 挂上去试试，毕竟这层基建在 2026.2.3 就躺在那儿了，只是没人告诉你。
