---
title: "DeepSeek Super-agent：五个模块只剩两个"
date: 2026-11-01T10:33:00+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent", "DeepSeek", "架构分析"]
summary: "官方文章讲了五个模块，我把仓库翻了一遍：真正落地的只有两个，其余靠已有机制兜。显式和隐式的差别，决定了这套架构能走多远。"
cover:
    image: "/images/dsh-superagent/cover.png"
    alt: "DeepSeek Super-agent：五个模块只剩两个"
    relative: false
---

2026 年 10 月 31 日，DeepSeek Harness 团队发了一篇讲 Super-agent 的长文。我没急着转，先把配套仓库拉下来翻了一遍。

文章讲的事很具体：人和 agent 的关系卡在注意力上。一个人顶多同时跟三到五个活跃会话，再多就顾不过来。所以他们想造一个常驻的、自己接活自己干的 agent，让人的角色从「每一步都盯着」退到「偶尔看一眼」。

四个特征：连续、高效、主动、持久。拆成五个模块：双层架构、事件驱动、面向人的交互、存算分离、Watchdog。

我的规矩是，官方说设计，我回代码里看。仓库 `deepseek-ai/dsh-superagent-prototype` 是当天下午五点半建的，现在两千九百星、两百五十三个 fork。`src` 下面九十二个 TypeScript 文件，一万两千三百九十行。

先说结论：五个模块，代码里能搜到两个半。

## 五个模块，代码里能搜到几个

我把文章里那几个专有名词一次性 grep 到 `src`、`docs`、`prompts` 三个目录。命中为零的就是没落地：投机执行、Inline 与 Spill、链式召回、端侧 daemon、Watchdog，全是零。

![五个模块的落地情况](/images/dsh-superagent/fig1.png)

这个落差不是作者藏了什么。README 自己写着：

> We have adapted it for publication, so it is not identical to our internal version... Some designs discussed in the article are directions for future development and are not implemented in this prototype.

意思是公开发布做过裁剪，文章里有些东西属于未来方向。真实范围是：盯一个 GitHub 仓库，可选加一个飞书，有人提需求就判断接不接，接了在独立目录里干，干完回原会话答，通常开个 PR。

一句话，它现在是个只认 GitHub 的接单机器人。

## 骨架是五个部件，从收消息到回话一条直线

业务逻辑压在一个二百七十行的 `engine.ts` 里，内部拆成五块，依赖单向流动：Intake 收、Evals 判、Tickets 建单、Assignments 干、Replies 回。

Intake 收消息存一次，扇出给订阅这件事的工单。Evals 管评估，合并同一串讨论、算强度、定截止时间。Tickets 管生命周期，Assignments 收执行器回报，Replies 走 outbox 往信道写。

每个调用方只拿到自己那一片接口，收消息的碰不到工单表，执行器改不了策略。这个切法比功能本身更值得抄：它把「谁能改什么」写成了类型，越权在编译期就过不去。

![Engine 的五个部件和两个 preset 的权限边界](/images/dsh-superagent/fig2.png)

## Router 不是「不给 Bash」，是配置里没有写权限

文章里有个说法容易被读成比喻：Router 层不直接执行任务，连 Bash 都不配。代码里是两个 preset 实现的，Worker 有 subagent、todo、web 搜索、文件读写；Router 把文件写、编辑、subagent、web 全关了。

那个「不配」是真不配。权限面在服务装配那一刻就定死，不是靠提示词嘱咐它别动手。

## 有三处比文章写得更细

准入矩阵。谁来提需求算数，文章只说两级过滤加名单，代码里是一张四格表：名单内名单外，乘上有没有 @ 它，每格配成显式、推断、忽略三档。

截止时间是活的。一次评估到点不会直接丢，重排两次才记成不可用；计时从候选送进主会话那刻才开始，处理暂停时跟着顺延。

还有插队。执行器最多同时跑三个工单，人在会话里直接插话时可以多用一格。

这三样文章都没展开。写到这个粒度，说明作者真在拿它跑活儿。

## 装起来不难，难的是两个不许弄错的地方

装的部分很快。tmux 起服务，克隆，`pnpm install` 十八秒，`pnpm build` 一秒，产出十一个包共六百九十多 KB。然后从 web 模板复制一个 profile，把仓库挂成本地 link，写两行配置：状态目录和面板端口。我用了 3082，不跟日常那个 3080 抢。

起来之后七个部件全 ready，侧栏多出一个入口。到这一步你会以为成了。

## 第一个坑：它必须是另一个账号

这东西要求 agent 用一个独立的 GitHub 账号发评论、开 PR，官方部署文档写得直白：agent 写的每句话都会挂在那个名字下面。我第一次授权失败得很典型，浏览器那边点了通过，我这边换 token 时直连 github.com 超时，二十秒的窗口过去，那次授权就作废了。换成走代理重来一遍才成。

连带一个问题：它干活时 `git push` 也要出网，但服务不能在全局环境挂代理，否则 DeepSeek 的接口会被带偏。我的做法是代理只注入两处，一个 GitHub CLI 包装脚本，一个工作克隆的仓库级 git 配置。

## 第二个坑：全部 ready，它却一个字都不说

这个坑贵得多，而且它一声不吭。

配置填完，面板没有报错，七个部件全绿。我发了个测试请求，到达数加了，评估数一直卡在一。

翻会话记录才看到真话：`llm-deepseek: no API key for provider route "deepseek-official"`，错误码 `MISSING_CREDENTIAL`。

![面板显示全部 ready，而 Router 会话里每一轮都在失败](/images/dsh-superagent/fig3.png)

根因是环境。日常那个 dsh 进程从一个 export 过密钥的 shell 里起来，新 profile 是干净环境，密钥只存在那次 export 里，不写文件。

面板看不到，因为这是 Router 内部会话失败，不算组件健康问题。整条链路每环都 ready，只有最中间那次模型调用在静默地死。

修法是加一层启动包装，从密钥该在的地方取出来再注入。代价是以后启停必须走那个包装，直接调官方脚本会把密钥丢掉，然后你又回到「一切正常但什么都不做」。

## 装完之后，我在 Issue 里等它回话

真验收是让它干件小事。我开了条 Issue，只要求回一句已连通、说出 README 第一行的标题，不许改文件不许开 PR。

不到十分钟，它以 `dsh-super-agent` 的身份回了：

> 已连通。
> README 第一行标题：`# ADK for Kotlin + Ktor：KMP 开发者的 AI agent 起手式`

回复上面还自带一行免责声明，说这是自动程序写的，不是人。

![Issue #1 里它的回复实录](/images/dsh-superagent/fig4.png)

我回头看它干了什么：给这条讨论建工单，在状态目录下开了独立 worktree，分支名 `agent/冒烟测试...`，干完在同一个讨论里答话。紧接着它把第二条重复请求判成重复，没再回一遍。

这一步之后我才信它真在跑，而不是看起来在跑。

## 现在停在观察模式，还没敢让它自己开工

四路观察我都打开了：新 Issue、新 PR、新评论、默认分支的 CI。但评估模式还是默认的 observe，意思是它只记「我本来会接」，不动手。理由很简单，两个 preset 都是满权限，以我本人的系统用户跑，能读我整个 home。任何一句进了 GitHub 评论的提示词注入，都能在我机器上执行命令。

所以顺序是：先跑一两天，翻它的判断记录，看有没有把不该接的活接进来。名单里只有我一个人，别人 @ 它不会有反应。

文章里有个数据我印象很深。去掉订阅直通之后，Router 的决定数涨了 30%，token 涨了 37%。作者拿这个说过滤层的价值。

我倒觉得这是全篇最该记住的一句：这类系统真正的成本不在模型跑多快，在它要不要为每件事都动一次脑子。

那么问题来了，一个自己接活、自己判断、权限跟你一样大的 agent，你愿意让它替你收一晚上的 GitHub 通知吗？

还是说，你更想要的只是一个能替你盯着、但每次动手前都要点头的工具？

如果你也在搭这类东西，说说卡在哪一层：安装、权限，还是那句「它到底该不该自己决定」。
