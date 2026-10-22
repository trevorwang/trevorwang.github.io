---
layout: post
title: "EmbeddingGemma 2：一个模型吃掉四种模态"
date: 2026-10-10 08:40:26 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/embeddinggemma2/cover.png
---

![EmbeddingGemma 2：一个模型吃掉四种模态](/assets/images/embeddinggemma2/cover.png)

在端侧做搜索，过去是个拼装活。图片要先用描述模型转成文字，语音要先转写，视频要抽帧再加字幕，最后统统丢给文本嵌入模型。每加一种模态就多一条链，多一份延迟，多一堆中间产物。

10 月 6 日，Google DeepMind 发布了 EmbeddingGemma 2，他们把这条链收进了一个模型。

## 它是什么

740M 总参数，拆成三块：文本加代码主干 270M，视觉编码器 170M，音频编码器 300M。四类模态（文本、图像、视频、音频）投影进同一个 768 维空间，还能交错混排，用占位符标记媒体位置就行。

上下文窗口 8192 token，是上一代的 4 倍。单次输入最多塞进 5.5 分钟音频、29 张图或者 58 帧视频。

许可用的是 Apache 2.0，官方说架构基于 Gemma 4、和 Gemini Embedding 同源技术。上一代 EmbeddingGemma 累计下载超过 2000 万次，这是它敢一次把模态铺开的底气。

## 四个配置，共用一个向量空间

这个模型最值得说的是模块化。你可以只加载需要的编码器：

```
只用文本        270M
文本加图像      440M
文本加音频      570M
全模态          740M
```

关键在最后一句：四个配置共用同一个向量空间。用 270M 编码的查询，可以直接和 740M 全模态编码的文档比相似度。也就是说你不必一开始就下决心，先用纯文本建索引，哪天想加图片检索，把视觉编码器加进来就行，已经算好的向量不用重算。

![](/assets/images/embeddinggemma2/fig1.png)

## 存储能省六倍

输出是 768 维，但它用了 Matryoshka 表示学习，可以把向量截短到 512、256、128 维，存储最多省 6 倍。

代价写在官方的表里：256 维基本保住文本和代码的质量，多模态检索大约 95%；128 维文本和代码还剩约 90%，但图像、视频、语音会掉到 75% 左右。官方给的建议很清楚，128 维留给大索引和粗排，别直接拿它做多模态检索。

## 代码能力这次涨得最狠

多语言文本几乎没有变化，真正的大跳在代码上：

```
MTEB 多语言 v2     61.36   （上一代 61.15）
MTEB 代码 v1       78.68   （上一代 68.76）
```

代码这一项涨了 9.92 分，约 14%。官方把用途写得很直白，本地代码库索引、语义代码搜索、给编码 agent 做检索。

其他模态的分数也一并列出来，都是官方口径、768 维：

![](/assets/images/embeddinggemma2/fig2.png)

## 端侧到底有多小

官方给的实测口径是，在 Pixel 11 Pro 上量化之后，纯文本权重约 191MB 活跃内存，全模态约 567MB。加上量化还能再压，Unsloth 那边甚至在提 0.5GB 内存里跑。

配 Gemma 4 用还有个隐性好处：两个模型共用同一个文本分词器和音频编码器，一起跑端侧 RAG 时总内存比各跑各的要低。

## 三个会静默失效的坑

这几条都写在官方模型卡里，值得单独拎出来，因为它们不报错。

第一个是精度。不要用 float16。模型的激活范围超出 fp16 的动态范围，结果是返回 NaN 或者悄悄劣化的向量，不抛异常。用 bfloat16，没有原生支持的硬件用 float32。

第二个是截断。切片会破坏单位长度，截短之后必须重新做 L2 归一化。不归一化不会报错，只会让排序质量慢慢变差，分数看着还挺像样。

第三个是维度对齐。查询和文档必须同一个维度，768 维的查询不能拿去跟 128 维的语料打分。

另外文本输入要带任务前缀，检索用 SearchQuery 加 Document，代码检索用 CodeRetrieval，分类和聚类用各自的对称前缀。前缀省了也能跑，只是质量会掉。媒体输入不加前缀。

## 生态这次铺得很快

发布当天基本就绪的工具链有：sentence-transformers（6.1.0 起）、transformers、vLLM、SGLang、MLX、Ollama、LM Studio、Qdrant、LiteRT。

llama.cpp 的 GGUF 版本在 ggml-org 仓，微调有 Unsloth 的文档，浏览器侧有 transformers.js 和 WebGPU demo。

落地形态上，Google AI Edge Gallery 加了两个演示。一个是 Instant Media Search，边打字边搜本地相册，输入德语的「Katze schläft auf Tastatur」能直接命中猫睡在键盘上的那张。

另一个是 Video Moments Finder，用「小孩在笑」这种描述直接定位到视频里的时间点，不需要转写音频也不需要生成字幕。

Mac 上还发了 AI Edge Foresight，本地会议助手，靠 EmbeddingGemma 2 做文件检索、Gemma 4 做理解，接系统音频和麦克风，离线也能用。

再往后，ML Kit 会在接下来几周提供这套能力，带 NPU 加速；MediaPipe 的 UniversalEmbedder 和 SemanticRetriever 已经支持，顺手还多了一个用途：不做任何微调，把用户输入直接跟分类标签比相似度，就能做出毫秒级的意图路由。

## 一句判断

这条消息真正值钱的地方不是又刷了哪个榜，而是一个 740M 的开源模型替掉了三段式流水线。省下来的不只是内存，还有中间产物和那些串联的延迟。

对端侧 RAG、本地文件检索、代码库索引这三个方向的人来说，这是今年值得花一个下午试一次的东西。要注意的是，HF 主仓里没有 GGUF 和 ONNX，做端侧部署得去 ggml-org 和 litert-community 两个仓拿。

你会把哪个场景搬回本地跑，评论区聊聊。

参考：
- 官方发布博客 https://blog.google/innovation-and-ai/technology/developers-tools/embeddinggemma-2
- 模型卡 https://huggingface.co/google/embeddinggemma-2
- 开发者指南 https://developers.googleblog.com/en/embeddinggemma-2-the-developer-guide/
- AI Edge 落地篇 https://developers.googleblog.com/google-ai-edge-with-embeddinggemma-2/
