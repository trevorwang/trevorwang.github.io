---
title: "最懒的那个资深开发，正在给 AI agent 立规矩"
date: 2026-10-05T07:57:08+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "你让 agent 加个日期选择器。它装一个日历库，写一层 wrapper 组件，补一个样式表，然后开始跟你讨论时区怎么处理。"
cover:
    image: "/images/lazy-agent/cover.png"
    alt: "最懒的那个资深开发，正在给 AI agent 立规矩"
    relative: false
---

## 先说那个熟悉的人

你让 agent 加个日期选择器。它装一个日历库，写一层 wrapper 组件，补一个样式表，然后开始跟你讨论时区怎么处理。

这时候你会想起办公室里那个人。马尾辫，椭圆眼镜，在公司待得比版本控制还久。你把五十行代码递过去，他看一眼，什么也没说，回来改成了一行。

ponytail 这个项目干的事，就是把他塞进你的 agent 里。它 15.5 万星，MIT 协议，README 第一行写的是：他什么也不说，他写一行，它能跑。

## agent 天生爱做加法

agent 的默认动作是加法。你不拦它，它就给每个需求配一套新东西：新依赖、新抽象、新样板。这事不能全怪它，训练数据里那些工程化做法确实长这样。

问题是它不心疼你的仓库，也不心疼你的账单。上面那个确认框，你要的就是一句「确定要删吗」，而浏览器从 2022 年起就有原生的 `dialog` 了，焦点管理、Esc 关闭、遮罩层都是白送的。

热榜上这两天的「少写代码」一派，讲的就是这件事。

## 它的全部家当，是一份 2.5KB 的规则

ponytail 的核心不是代码，是一份 2,492 字节的规则文件。里面是一条阶梯，从第一级往上爬，哪一级站得住就停在哪：

```text
1. Does this need to be built at all? (YAGNI)
2. Does it already exist in this codebase?
3. Does the standard library already do this?
4. Does a native platform feature cover it?
5. Does an already-installed dependency solve it?
6. Can this be one line?
7. Only then: write the minimum code that works.
```

> 译文：① 这东西真的需要存在吗？② 仓库里已经有现成的吗？③ 标准库能做吗？④ 平台原生能力覆盖了吗？⑤ 已经装过的依赖能解决吗？⑥ 能写成一行吗？⑦ 到这一步，才写「能跑的最小代码」。

阶梯前面还压着一句话：先把任务和它碰到的代码读完，把真正的流程从头到尾走一遍，再开始爬。顺序反了就不是懒，是拿效率当借口。

## 我把它跑起来了

这份规则被撒得很宽，Claude Code、Codex、Cursor、Windsurf、Copilot、Hermes 都各有一份适配文件。我挑了最干净的一条路，它自带一个 MCP server：

```bash
cd ponytail-mcp && npm install && node index.js   # 用 stdio 讲 MCP
```

走一遍真实的握手，它给我的东西是这样：

![](/images/lazy-agent/mcp-handshake.png)

三档强度（lite / full / ultra）的正文只差几十个字符。也就是说强度调的是语气和尺度，不是提示长度，那份规则不管哪一档都在，约 5,200 字符。

它 README 里还写了个容易被忽略的细节：走 MCP 这条路，规则是按需拉的；想让它每一轮都生效，得走 hook 或者插件那条路。别看见 MCP 就以为已经接管了，得看你接的是哪一头。

## 省下来的数字，和它没省的

README 里的数据是真跑出来的。拿一个真实开源仓库当靶子，同一个 agent 装和不装各跑十二个功能任务，比它留下的 git diff：代码量少 54%，token 少 22%，成本少 20%，时间少 27%，安全检查全过。

几个具体例子，同一个模型、同一个提示词的对照：

- 搜索框防抖：116 行 → 10 行
- CSV 求和：20 行（上 pandas）→ 3 行（csv 标准库）
- 删除确认框：1 个依赖 + 30 行 → 0 个依赖 + 8 行（原生 dialog）

然后是我觉得最该看的一列，对照组。只跟它说「写得越短越好」，代码能少 33%，但安全检查掉到 95%。换个话说得少、却没规矩的写法，代码少 20%，token 反而多 7%，成本多 3%。

所以少写本身不值钱，值钱的是它同时写下的另一份清单，不许懒的地方：理解问题、信任边界上的输入校验、防数据丢失的错误处理、安全、可访问性、真实硬件的校准。再加一条，非平凡的逻辑要留一个能跑的检查，那种一旦逻辑坏掉就会失败的检查。

还有一处挺难得：它自己在 README 里把数字往低调了。早先单轮基准报的是 80% 到 94%，后面写明，对着更公平的基线跑，那是单个任务的上限而不是平均值，均值是 54%。

## 另一条路是加法

这两天跟它挨着上热榜的还有一个 ECC，星数比它还高，走的是完全相反的方向。仓库里 4,212 个文件、1,027 个 SKILL.md、README 十万字符，它自称提供 293 个技能和 68 个 agent。

它不教 agent 少写，它把技能、本能（instincts，带置信度，从真实会话里学）、记忆、安全扫描、调研先行的流程全装进 harness。口号很诚实：优化上下文窗口，其余全部持久化。

![](/images/lazy-agent/two-camps.png)

两派的分歧不在懒不懒，在你把预算花在哪。一个花在少产生代码，一个花在多记住事情。ponytail 整个仓库 1.78MB，ECC 是 56MB。

## 落到我自己这台机器上

我平时用的 agent 客户端是 DeepSeek Harness（dsh），口号是 Everything is a Plugin。我在里面挂 MCP 的方式就是这么一段：

```yaml
- insert:
    - id: memory-memorix
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: memorix
        transport: stdio
        command: memorix
        args: [serve, --mode, lite]
```

ponytail 的 MCP server 讲的是同一套 stdio 协议，要接进来就是把 command 和 args 换掉。接之前先想清楚一件事，这类东西一旦走插件或者 hook 注入，它就进你每一轮的上下文，token 是你要付的。

## 我现在的判断

写在一张清单上，你可以直接对着看：

- 它省的是「写」还是「返工」。少写代码不等于少踩坑，返工率不在它的指标里。
- 规则进不进每一轮。MCP 是按需拉，插件和 hook 是每轮注入，两条路的账不一样。
- 你的项目里哪些不许省。写下来，比说一句「别啰嗦」有用得多。
- 加法那条路也没错，但先算上下文账。

回到那个马尾辫的同事。他让人服气不是因为他写得少，是因为他知道多出来的每一行，将来都得有人维护。给 agent 立规矩也一样，重点从来不是让它少打字，是让它知道哪些东西碰不得。

你手上那个项目，agent 最近给你塞过最没必要的是什么？评论区聊聊。

参考资料：
- https://github.com/DietrichGebert/ponytail
- https://github.com/affaan-m/ECC
