---
title: "JetBrains 的新 IDE，编辑器是只读的"
date: 2026-11-04T21:10:00+08:00
draft: false
categories: ["公众号"]
tags: ['JetBrains', 'AI IDE', 'ThinkRail']
summary: "JetBrains 出了个 AI IDE，我把它跑起来才发现关键不在功能表：它源码里写死 readOnly: true，我试着往编辑器里打字，文件哈希一动不动。这是一个不让你改代码的 IDE。"
cover:
    image: "/images/thinkrail/cover.png"
    alt: "JetBrains 的新 IDE，编辑器是只读的"
    relative: false
---

JetBrains 在 2026 年 8 月 19 日发了个新东西，叫 ThinkRail，中文圈几乎没人写。

我先看了仓库再装的本机版。看完源码有一个发现，比它所有功能都让我意外：这个「IDE」的编辑器是只读的。

`monacoSetup.ts` 第 110 行写着 `readOnly: true`，还配了一句提示文案。我打开界面试着往里敲字，一个字都进不去，文件哈希也没变。

这篇文章就讲这件事：一个不让你改代码的 IDE，到底在赌什么。

## 它不是 IntelliJ 的下一代

仓库在 `JetBrains/thinkrail`，Apache-2.0，README 上挂着「JetBrains incubator project」的徽章。

官方对自己的描述只有一句：在这个轻量 IDE 里用 pi 来做 vibe coding，它会围绕你的工作方式自我定制。

pi 是什么？一个第三方的极简 agent harness，包名 `@earendil-works/pi-coding-agent`，作者是 Earendil 那帮人。它的哲学是「不给功能，给原语」，子 agent、plan mode、权限弹窗全都不内置，让你自己写扩展。

ThinkRail 干的事，是给这个命令行 agent 套一个完整的界面：工作区、编辑器、diff、终端、spec 树。

所以你看到的东西是：一个两千多人的公司，用 TypeScript 写了一个壳，去包一个别人写的 agent。这跟 IntelliJ 那种自研内核的路子，是两条路。

![](/images/thinkrail/fig1.png)

## 装起来是二百一十八兆的一个文件

它有两种形态：桌面安装包，和自包含的命令行版本。

我下的命令行版，单文件 218 MB，里面塞了 Bun 运行时，不用你先装 node。跑起来是这样：

`./thinkrail --no-open --port 24242 <你的仓库路径>`

浏览器打开 `localhost:24242` 就是完整界面。它没有自己的账号体系，模型走 pi 那套配置。

我本来准备去设置里填 DeepSeek 的 key，结果一打开 Providers 页就看见一行：DeepSeek，环境键 `DEEPSEEK_API_KEY`，已连接。

它直接读了我 shell 环境里 pi 已经配好的那套连接，零配置。

## 我给它一个坏掉的仓库

测试题目我特意准备了：一个小 JS 库，把标题转成 URL slug，两条测试里有一条是红的，因为实现没处理标点。

入口页有三个选项：写 specs、直接开工、在项目目录里干。我点了第一个，那是它的招牌流程。

提示词我写得很克制：按现状把 specs 写出来，先不要改代码。

点创建之后，它先做了一件事：在 `~/.thinkrail/worktrees/demo-repo/workspace-1` 切了一个真的 git worktree，分支叫 `write-slug-lib-specs`。我回主仓看了一眼，`main` 还在原来的提交上，一根毛都没动。

这一步不是模拟的，`git worktree list` 能看到两行。

## 六分十七秒，三个文件

跑完我看结果：6 分 17 秒，26 次工具调用，15 个步骤，其中 `spec_create` 三次、读文件三次、改文件三次。一兆的上下文只用了 5.3%，花了 0.138 美元。

它没有直接开写。先 `ls` 看结构，读 README，读测试，然后真的跑了一遍 `node --test`，看到那条测试红了，才开始动笔。

产出三个 markdown：

`goal-and-requirements.md` 讲这个库是什么、给谁用、现在只做「小写化 + 空格换横线」。`architecture.md` 讲单模块拓扑、依赖只有测试指向源码。`src/SPEC.md` 讲 `slugify` 的契约和边界。

三个文件都带 frontmatter，`id`、`type`、`status: draft` 加 `parent`，串成一张图。不是我教它的格式，是它载入的 `spec-graph` 技能包里定死的规矩。

最有意思的是它对那条红测试的处理。它没有把「会剥标点」写成已实现的能力，而是在模块 spec 里单开一节「Known limitation」，写清楚：测试期望 `hello-world`，函数返回 `hello,-world!`，标点原样留着。

做完它自己跑了一次 `spec_validate`，通过。

![](/images/thinkrail/fig2.png)

## spec 这张图长什么样

三个文件不是并列的文档，是有 `parent` 指针的树。

根是目标文档，中间是架构，叶子是具体模块的 spec。界面上右边那栏就是这棵树，点一下某个节点，左边编辑器直接渲染对应文件，还有 Preview 和 Source 两个视图。

它的博客在那篇《Spec-Driven Development》里写道，这么做的目的是让意图活过会话。聊天关了，代码还在，但「为什么这么写」通常跟着聊天一起消失。spec 就是把这部分留下来，而且 agent 每次开工都先把这张图读进去。

我这次只让它写，没让它改。真要改的时候，它读的是同一份图。

![](/images/thinkrail/fig3.png)

## 然后我发现了只读这件事

写完之后我想自己动手改一句，点进 `src/SPEC.md` 的 Source 视图，选了一行按键盘。

没反应。我以为是焦点问题，点了几次，又用开发者工具直接往可编辑区灌文本。结论一样：输入不进去，磁盘上文件的哈希值前后完全一致。

回到源码找答案，就是开头那句 `readOnly: true`。界面上甚至准备了提示语，告诉你去让 agent 改。

这条在 Hacker News 上有人提过，说 ThinkRail 里聊天面板是主角，编辑器和文件树是配角，「你甚至不被允许手动改文件」。

我核了一遍，属实。但要说公平：终端是能用的，底部就有一个真的 PTY，你在里面 `vim` 谁管不着。被锁的只是那个 Monaco 面板。

问题是这个设计把「改代码」这件事的主语换了。以前的 IDE，主语的你和 agent 各占一半。这里你和 agent 的分工很清楚：agent 写，你审。

![](/images/thinkrail/fig4.png)

## 它为什么敢这么设计

因为它的架构里，IDE 那部分本来就不是重点。

官方架构文档把整个系统拆成三个环：引擎宿主、协议线、界面客户端。

第一个环跑在 `packages/server` 里，用 Bun 起一个 HTTP + WebSocket 服务，pi 在进程内跑。第二个环是 `packages/contracts` 里那份带版本的协议，它是两端唯一的耦合。第三个环是 `apps/web`，一个 React 写的、移动优先的客户端。

「移动优先」这四个字不是修辞。文档里写了：UI 独立于宿主发布，通过网线拨号到某个宿主，手机可以经 Tailscale 连上来。

所以它从第一天起就没把自己当传统 IDE 设计。它是一台 agent 主机的远程面板，桌面版只是顺便用 Electrobun 打了个壳。

编辑器只读不是阉割，是这个架构的自然结果：界面只是观察者和审批者，真正动手的是宿主里的 agent。

![](/images/thinkrail/fig5.png)

## 仓库本身

代码规模不小：TypeScript 和 TSX 一共 1,493 加 349 个文件，35.4 万行。

技术栈也很新：Bun 运行时、React 19、Monaco 0.57 加 shiki、xterm 6、Tailwind 4、TypeScript 6.0.3，网站用 Astro。桌面壳不是 Electron，是 Electrobun。

社区数据：7,484 星、450 个 fork、365 个 open issue。仓库是 2026 年 7 月 8 日建的，不到四个月做到这个数，节奏算快。

发版也密：0.2.0 是十月十八号，0.3.0 是十月二十九号，0.3.1 是十一月一号，夜版已经跑到 0.4.0。

## 两个不好听的地方

第一，遥测默认开着。`config.json` 里 `analyticsEnabled` 和 `analyticsConsentConfirmed` 都是 true。启动参数里那个 `--no-analytics` 关的是「额外用量数据」，基础统计不在里面，得进设置里关。

第二，JetBrains 的 AI 战略现在有点散。Hacker News 上那两条帖子一共两百多个点、两百多条评论，最高赞的那条不是夸它，是吐槽：Air 和 ThinkRail 都还是半成品，而真正好用的 AI Assistant 反倒被搁在一边。

这条我认同。ThinkRail 更像一次押注实验，JetBrains 自己都挂着 incubator 的牌子。

## 我的判断

如果你的第一反应是「IDE 不给编辑有什么用」，那它确实不是给你做的。

它服务的是另一批人：已经习惯让 agent 干活，主要工作在审查 diff 和写 spec 上，一个会话接一个会话地推进。对这批人来说，能编辑反而是个负担，因为你和 agent 抢同一个文件的时候，冲突只能靠自律解决。

它给的解法是物理隔离：每个工作区一个 worktree、一个分支，agent 的乱动留在它自己的分支上，你审完了才合并。

这个思路我觉得比「给编辑器加个 AI 按钮」更彻底。它赌的是：未来的 IDE 主要用来审，不是用来写。

赌得对不对，一年后再看。现在这一版，我会拿它跑点小活，不会拿它替 IDEA。

你平时写代码，改文件那一下是你自己动手多，还是已经交给 agent 了？如果已经是后者，这个只读设计大概不会让你难受。
