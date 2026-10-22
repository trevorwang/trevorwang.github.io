---
layout: post
title: "CLI-Anything：让 agent 用上命令行"
date: 2026-10-16 13:32:00 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/cli-anything/cover.png
---

![CLI-Anything：让 agent 用上命令行](/assets/images/cli-anything/cover.png)

## agent 点鼠标这件事，本来就不太对

让 agent 去操作软件，眼下最常见的路子是让它看截图、找按钮、点坐标。跑是能跑通，但很脆。窗口大小变一变、按钮挪一格、页面慢半拍加载，它就开始瞎点。

港大的一个团队把这条路子的毛病写得很直白：

> Current GUI agents struggle with brittle pixel-level interactions, timing dependencies, and coordinate-based actions that break with interface changes.

大意是：现在的 GUI agent 卡在脆弱的像素级交互、对时序的依赖，以及一改界面就失效的坐标动作上。

他们给的替代方案朴素得有点意外，别让 agent 看屏幕了，给软件配一套命令行。

## CLI-Anything 把这件事做成了流水线

项目叫 CLI-Anything，挂在港大数据智能实验室（HKUDS）名下，背后有一篇论文《CLI-Anything: Towards Agent-Native Computer Use》。10 月 16 日我看的时候，53,122 个 star，Apache-2.0，Python 写的。

它交付的东西是一个仓库加一个 hub。hub 本身装起来就一条命令：

```bash
pip install cli-anything-hub
```

装完能挑的东西是 103 个命令行工具，其中 79 个是它们自己为具体软件写的 harness，24 个是第三方的现成包，盖住 35 个分类。挑中哪个，还是一条命令：

```bash
cli-hub install 3mf
cli-hub install ollama
```

命名是统一的，装完入口就叫 `cli-anything-3mf`、`cli-anything-ollama`。

它还有一层叫能力矩阵，5 张表按场景把 CLI 组织起来（视频 14 个、知识研究 13 个、游戏 9 个、图像设计 7 个、3D CAD 6 个），矩阵自带缺口检查和任务配方。问它「做一条短视频需要哪些工具」这类问题，它会列出每个能力点有谁提供、缺哪个。

## 装一个 CLI，agent 手上多三样

第一样是一份写给 agent 看的技能文件。每个 harness 里都有 SKILL.md，写清这个软件能干什么、命令怎么敲、有哪些坑。它还顺手给 Claude Code、Cursor、Codex、Hermes 各出了一份对口的技能包。

第二样是结构化输出。命令基本都能加 `--json`，返回机器好读的 JSON，而不是一屏给人看的排版文字。

第三样是会话状态。harness 是带状态的：有 `session` 一组子命令管状态，也有 REPL 交互模式。也就是说，agent 可以像人一样一步一步来，不必每次把上下文重讲一遍。

![CLI-Anything 交付的三层](/assets/images/cli-anything/fig1-three-layers.png)

## 上手实测：三个软件，三种收场

我在自己这台 Mac 上真跑了一遍，挑的三个都是装得出来、又能真出结果的：3MF、Ollama、LLDB。

先说 3MF，这是个 3D 打印文件格式，harness 干的事是找圆柱孔、改孔径、修网格。我用 trimesh 造了一块 20×20×10 的板子，中间一个贯穿的 3.0mm 圆孔，然后让 CLI 自己找：

```bash
cli-anything-3mf --json inspect --axis 2 plate-with-hole.3mf
```

```json
{
  "hole_count": 1,
  "holes": [
    {
      "hole_id": 0,
      "center": [-5.09e-18, -9.34e-06],
      "diameter": 2.9988,
      "axis": 2,
      "confidence": 0.9996,
      "vertex_count": 128
    }
  ]
}
```

真值是 3.0mm，它报 2.9988，误差 4 微米。接着把孔改到 4.2mm：

```bash
cli-anything-3mf resize -h 0 -d 4.2 -o plate-4.2.3mf -a 2 plate-with-hole.3mf
# Resized 1 hole(s) to 4.2mm
# Hole 0: 2.999mm -> 4.200mm (128 vertices)
# Saved: plate-4.2.3mf
```

耗时 0.33 秒。改完再核一遍，报的直径是 4.1983。最后让它比对新旧两个文件，体积从 3929.43 立方毫米降到 3861.68，少了 67.75。

这个数能自己验：把孔径从 3.0 换到 4.2、板厚 10，理论挖掉的是 π/4 × (4.2² − 3.0²) × 10 = 67.86 立方毫米，和它给的 67.75 差 0.16%，正好是 128 边多边形近似圆的误差。

![3MF 实测闭环](/assets/images/cli-anything/fig2-3mf-loop.png)

再说 Ollama。这台机器上本来就装着 Ollama，有两个模型，一个 bge-m3（嵌入）、一个 gemma4（8B 的对话模型）。harness 包装的正是本地这台服务：

```bash
cli-anything-ollama --json model list      # 列出本地模型，带 digest、量化等级
cli-anything-ollama --json embed text -m bge-m3 -i "把软件包成命令行"
# → 1024 维向量，模长 0.999999，耗时 1.77 秒

cli-anything-ollama --json generate text -m gemma4 -p "用中文两句话解释……" --no-stream --num-predict 1024
# → 647 个 token，22.99 秒，返回一段真中文
```

这里踩到一个值得说的坑。第一次我只给了 60 个 token 的上限，返回的 `response` 是空字符串，看着像模型没干活。但同一份 JSON 里写着 `eval_count: 60`、`done_reason: "length"`，说明它确实生成了 60 个 token，只是全花在内部思考上，没轮到正文就撞上限了。

把上限加到 1024，正文就出来了。换谁写 agent 都该记住这条：本地模型返回空字符串，不一定是没生成，得看 `done_reason` 和 `eval_count` 这两个字段。

![同一个模型，两种 token 上限](/assets/images/cli-anything/fig3-token-limit.png)

第三个是 LLDB，调试器。它装得上，但在这台机器上真调不起来，因为系统那份 lldb 卡在 Xcode 许可门后面。有意思的是它失败的样子：

```json
{
  "error": "LLDB not found. Install LLDB:\n  macOS:  xcode-select --install\n  ...",
  "type": "RuntimeError"
}
```

退出码 1。它没给我一坨堆栈，而是把探测到的 stderr 原文、平台的安装命令、错误类型一起塞进 JSON 里。这种失败方式对 agent 是友好的，读一个字段就知道下一步该干什么。

三个跑完，对照是这样：

| 软件 | 装法 | 实际跑出来的结果 |
|---|---|---|
| 3MF | 本地装 harness | 找到 2.9988mm 的孔，改成 4.2mm，体积账对得上 |
| Ollama | 本地装 harness | 1024 维向量、647 token 真生成，另有一个空响应的坑 |
| LLDB | 本地装 harness | 没跑起来，但给出结构化 JSON 与安装指引，退出码 1 |

## 上手门槛和几个坑

装的过程比我预想的费劲，坑都集中在这一段。

`cli-hub install` 内部是调 pip 装包，所以 hub 自己所在的 Python 环境里必须有 pip。用 `uv tool install cli-anything-hub` 装（好处是不污染全局），环境里没有 pip，这条子命令就直接报 `No module named pip`。想让官方流程能用，就建个带 pip 的虚拟环境再装。

它从 git 装包，代理环境下会撞两次网络问题：先是 `Error in the HTTP2 framing layer`，把 git 的 http.version 临时降到 1.1 之后，又变成 `SSL_ERROR_SYSCALL`，线路本身在抖。稳妥的做法是绕开 git，直接下仓库压缩包，再本地安装：

```bash
curl -L https://codeload.github.com/HKUDS/CLI-Anything/tar.gz/refs/heads/main -o repo.tgz
tar xzf repo.tgz && cd CLI-Anything-main
python -m pip install ./3MF/agent-harness
```

macOS 上还有一层：/usr/bin/git 是 Xcode 的许可门挡着的 shim，pip 一调 `git version` 就以 `exit code 69` 挂掉。Xcode 包内那份 git 是能用的，把它挂到 PATH 前面就行。

用的时候也有两个小别扭。`--json` 是全局选项，得写在子命令前面（`cli-anything-3mf --json info x.3mf`），写后面会报未知选项。

还有 3MF 的找孔，不给 `--axis` 时在我这块板上返回 `hole_count: 0`，点明 `--axis 2`（孔沿 Z 轴）才认出来；改孔时要和找孔用同一套参数，否则 hole 编号对不上。

能力矩阵那层有个能力查询：`cli-hub can "transcribe audio"`。自然语言的问法不一定命中，实测这句返回的是 `No capability matches`，得按它自己的词表问。

## 那 MCP 呢

讲到这儿容易滑到一个结论：命令行要取代 MCP。它们自己的做法不是这样。

翻它的目录会发现，几个 harness 内部包的就是 MCP 服务器。抓文献的 zotero 那份，描述里写的是 CLI 与 MCP 服务器双形态，52 个 MCP 工具加 70 多条 CLI 命令；操作 Safari 的那份包的是 safari-mcp；浏览器那份包的是一个叫 DOMShell 的 MCP 实现。

我的理解是角色分了层：MCP 解决的是「服务怎么把能力暴露出来」，命令行解决的是「agent 怎么把动作敲出去」。前者要在进程外立一个服务、连一次、把工具表读进来；后者就是一条命令加一段 JSON。这个项目押的是后者的通用性，连它自己的 103 个 CLIs 也是靠 pip 这条最老的通道分发的。

两条路谁更省 token、谁更不容易出错，得真跑一轮对比才敢下结论，这篇先把它的链路走通。

你会先把哪个软件包成命令行？我盯上的是那些只有 GUI、没有批处理接口的老工具。欢迎评论区聊聊。
