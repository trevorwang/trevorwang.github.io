---
title: "Google ADK vs JetBrains Koog：ChatBot 后端该被谁选中"
date: 2026-10-31T11:18:00+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent", "Google ADK", "JetBrains Koog"]
summary: "同样要搭一个 ChatBot 后端，Google 官方的 ADK 和 JetBrains 的 Koog 押的是不同方向。六件事逐项对照下来，真正决定选谁的只有最后那两条独有能力。"
cover:
    image: "/images/adk-vs-koog/cover.png"
    alt: "Google ADK vs JetBrains Koog：ChatBot 后端该被谁选中"
    relative: false
---

要给 ChatBot 挑一个 Kotlin 后端，翻来覆去就是这两个库。

一个是从 Google 那套 ADK 搬过来的，一个是 JetBrains 自己做给 JVM 的。宣传页都能看，真到写代码，差别在哪儿得往下翻模块。

我把两个仓库都拉到本地，照着源码和模块清单，一项一项对。下面六件事，是 ChatBot 后端真正会碰到的。

## 第一件事：怎么描述一个 agent

ADK 记的是老账。Python 版那套 LlmAgent、tools、session、artifacts，Kotlin 版照搬，索引名、类名、参数几乎一比一。你在 Python 里怎么写，在 Kotlin 里换个语法接着写。

它有四种 agent 类型：LlmAgent 让模型自己决定下一步，SequentialAgent 按顺序跑，ParallelAgent 并发跑，LoopAgent 循环。多 agent 之间靠 TransferToAgentTool 移交控制权，靠 agent 树组织层级。

Koog 记的不是账，是图。节点、边、状态机，你自己拼。官方给了一堆现成节点：nodeLLMRequest 发一次模型请求、nodeExecuteTools 执行工具、nodeLLMSendToolResults 把结果送回去，光是这些节点就有二十六种变体，带不带工具、强制单工具、只调工具、多选、流式，各有各的函数。

这两种东西的差别在你想问题的方式。ADK 问「这人该派哪个 agent 去」，Koog 问「这一步走到哪个节点」。

![六件事逐项对照](/images/adk-vs-koog/fig4.png)

## 第二件事：工具

ADK 的工具是 Google 生态的富矿。GoogleSearchTool 直接搜，GoogleMapsTool 查地图，VertexAiSearchTool 接 Vertex 的检索，VertexAiRagRetrieval 接 RAG 库，UrlContextTool 读网页 URL。这些全是内置的，一行 new 就能用。

还有 FunctionTool 包你自己的函数，mcp 目录接 MCP 服务器，processor 模块用 KSP 注解处理器从带 @Tool 的函数自动生成工具定义，不用手写 JSON schema。

Koog 的工具更像一个开放的工具箱。agents-ext 里有一整套文件系统工具：ReadFileTool、WriteFileTool、EditFileTool、ListDirectoryTool、RegexSearchTool。

还有 ExecuteShellCommandTool 能跑 shell 命令。这一套下来，你拿 Koog 做一个 coding agent 的基础件基本齐了。MCP 也支持，单独有 agents-mcp-server。

差别在口味。ADK 的工具是「云上有什么」，Koog 的工具是「本地能干什么」。一个查 Google 地图，一个改本地文件。

## 第三件事：记忆和会话

ADK 是三层分得很清。Session 存对话，Artifact 存文件，Memory 存跨会话的知识。三样各有各的 service 接口，都有内存实现，也都留了换后端的口子。

会话还带事件压缩。summarizer 目录里有 EventCompactor、SlidingWindowEventCompactor、TokenThresholdEventCompactor，对话长了自动裁。

Koog 是往特征模块里塞。agents-features 下面挂着 memory、longterm-memory、chat-memory-sql、chat-history-jdbc、persistence-jdbc、snapshot 这些。要哪种记忆，就挂哪个模块。

历史压缩也有，DefaultHistoryCompressionStrategies 和 FactRetrievalHistoryCompressionStrategy。后者是压缩时按事实检索保留。

两边都有，但 ADK 是内置三层，Koog 是靠模块挑。

## 第四件事：失败之后怎么办

这是 Koog 最较真的地方，也是它 README 第三行就写的词：reliability and fault-tolerance。

内置重试不算什么，ADK 的 RetryConfig 也有。Koog 多了个 snapshot，agent 跑到某一点能把状态存下来，出事了从那个点恢复，不是从头再来。这套东西在长时间跑的任务里是真有用。

ADK 这边靠 ResumabilityConfig 和 workflow 里的 ResumeScan 做恢复，思路不同：它是从事件流里扫出断点，Koog 是把状态存成快照。

一个是重放日志，一个是存盘。看你信哪个。

## 走到这一步，说说 ADK 的几个细节

ResumabilityConfig 之外，ADK 还有几个东西值得单独提。

它有个 capabilities 层面的东西叫 AgentMode，能切 agent 的工作模式，配合 AgentModeTools 用。skills 目录很小，两个文件，是给 agent 装「技能包」的入口，Android 上从 assets 读。

plugins 里那个 ReflectAndRetryToolPlugin 挺实用：工具调用失败之后，它让模型看失败原因，决定要不要重试、换个参数还是换条路。这个不是简单的重试，是带反思的重试。

这几样 Koog 都没有现成的对应物。Koog 的做法是你在图里自己写一个判断节点。

## 第五件事：可观测

ADK 自带 telemetry 包，Span、SpanBuilder、TelemetryAttributes 一整套，还有 plugins 里的 LoggingPlugin、ReflectAndRetryToolPlugin。BigQuery 集成能把 agent 调用事件直接写进表里，这是它独有的，Google 云上做分析很顺。

Koog 走的是行业标准路线。agents-features-opentelemetry 导出 OTel，官方点名支持 W&B Weave 和 Langfuse 两个平台，事件处理器 agents-features-event-handler 可以挂自定义回调。

两边都能看到 agent 在干什么。ADK 的终点是 Google Cloud，Koog 的终点是你已经在用的那套观测平台。

## 第六件事：横切能力，这块 Koog 更全

ADK 有 callbacks，前后各三个点：before/after run、before/after agent、before/after model、before/after tool。要在这些点上插逻辑，得写 callback。

Koog 把它做成了 feature 体系，挂上去就行：event-handler 收事件，trace 记录调用链，tokenizer 算 token，opentelemetry 导出，acp 接 Agent Client Protocol，a2a 三件套做 agent 互通。想加什么能力，写一个 feature 挂上去，不用改 agent 逻辑。

这个设计差异挺大。ADK 是给你六个钩子，Koog 是给你一个插座。

## 那些其中一个有、另一个没有的

我列一下，这些是最能决定选哪边的。

ADK 独有这几样。

Java 一等公民，每个 builder 都有 Java 版，用 CompletableFuture 和 Reactive Streams 把协程挡在外面，不学协程也能写。

Android 端侧三件套：LiteRT-LM 设备上跑模型、ML Kit 调 Gemini Nano、Firebase AI 走云但不塞 API key。

dev UI 起服务就能看 trace 和 agent 图。A2A 和 agent 层级移交是原生设计。

Koog 独有这几样。

KMP 五端：JVM、JS、WasmJS、Android、iOS。ACP 协议支持，能接标准客户端。agents-planner 里有 GOAP 规划器。

agents-cli 直接做命令行 agent，还带 claude 和 codex 两个适配。文件系统工具全家桶，LLM 切换不丢历史。

![各自独有的，才是决定项](/images/adk-vs-koog/fig5.png)

## 拉回 ChatBot 这件事

上面六条是平铺着对的。真做一个 ChatBot 后端，六条的权重差得很远。

先说结论：ChatBot 最吃的是第二条之外的东西。

第一件事，描述 agent，对 ChatBot 来说没那么要紧。你的机器人多半只有一个主 agent，加几个工具，不需要复杂的 agent 树，也不需要二十六种节点变体。这块两边都远超够用。

第二件事，工具，是分水岭，方向还跟直觉相反。ChatBot 要的是查天气、搜网页、查地图、读链接。ADK 的 GoogleSearchTool、GoogleMapsTool、UrlContextTool 全是现成的，一行就能挂。Koog 的强项是读写本地文件、跑 shell，聊天机器人用不上这些。做 ChatBot，这条是 ADK 明显占优。

第三件事，记忆，ChatBot 的命。多轮对话要好，历史得管得住。ADK 内置三层加事件压缩，开箱就能撑住长对话。Koog 的功能都在，但要你自己挂模块、自己配。两边都能做到，ADK 省事，Koog 灵活。

第四件事，失败恢复。ChatBot 是长连接、多轮、随时可能断，这条不能忽略。Koog 的 snapshot 恢复更彻底，ADK 的事件流重放更适合已有事件日志的架构。谁的架构顺手就选谁。

第五件和第六件，可观测和横切，看你运维习惯。已经在用 OTel 和 Langfuse 的，Koog 接得干净。全在 Google 云上的，ADK 的 BigQuery 落表省一层。

所以 ChatBot 这个场景，功能表要换个权重再算一遍。

## 到底差在哪

写完这些，我发现一句话能概括。

ADK 是一整套云上方案在 Kotlin 里的端口。用它就得接受它的世界观：Gemini 优先、Vertex 认证、Session 和 Artifact 三层、agent 树、Google 的工具。做 ChatBot，这套东西开箱即用，你不用从零想架构。

Koog 是一个库，不是一个方案。它不规定你的 ChatBot 长什么样，只给节点、工具、特征三样东西，你自己拼。代价是架构得自己设计，好处是它不绑架你的技术栈。

所以选 ChatBot 后端，看的是三件事。

模型要不要用 Gemini，或者整套要不要落在 Google 云。要，ADK 顺。

机器人要不要查网页、查地图、读链接。要，ADK 的工具是现成的，Koog 得自己接。

团队是不是已经在跑 Kotlin 服务端，模型可能换、以后要上多平台。是，Koog 少给你很多约束。

我自己的项目在 Kotlin 服务端，接 DeepSeek，所以选 Koog。这个选择跟功能表无关，跟我的场景有关。

你要是做的是接 Gemini 的客服机器人，我大概会劝你直接上 ADK。
