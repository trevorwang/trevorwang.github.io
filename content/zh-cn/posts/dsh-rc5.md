---
title: "DeepSeek Harness rc.5：升级卡在两个包上"
date: 2026-11-19T06:54:00+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent", "DeepSeek"]
summary: "rc.5 只改了一条：DeepSeek 模型走服务端搜索。我从 rc.1 升上去，两次 install 都败在两个跟 rc.5 无关的包上，删掉它们才通。"
cover:
    image: "/images/dsh-rc5/cover.png"
    alt: "DeepSeek Harness rc.5：升级卡在两个包上"
    relative: false
---

rc.5 的更新说明只有一条。我把 dsh 从 rc.1 升上去，花了两个多小时，中间被两个包挡住。

这两个包都不是 rc.5 的问题，是它们自己停在了过去。

## rc.5 改了什么

官方 release 原文一句话：DeepSeek 模型使用 DeepSeek 服务端搜索，搜索不按次收费；其他模型默认使用 DeepSeek Search API。

发布时间 2026 年 11 月 18 日 12:51，归在「体验优化」下面。

这条改动本身值得注意。搜索不按次收费意味着，模型自己决定要不要查、查几次，都不会变成账单上单独的一行。以前这类工具调用是容易被反复计费的地方。

两种搜索不是一回事。DeepSeek Search API 是往外调一个检索服务，按次算钱，模型每查一次就记一次。服务端搜索是模型自己在服务端的工具环里检索，对使用者来说不计次。rc.5 把 DeepSeek 自家模型默认切到前者那条免费路上，别的模型仍然走 Search API。

## 但 latest 还指着 rc.1

我先去 npm 上看了一眼版本。

`@deepseek-ai/dsh` 的 dist-tags 是这样的：`latest` 是 `0.2.2-rc.1`，`next` 是 `0.2.2-rc.5`。

也就是说，一个人跑 `npm install -g @deepseek-ai/dsh`，装到的还是 11 月 4 日那个 rc.1。要拿到 rc.5，必须显式写 `@next` 或者写全版本号。

![npm 上 latest 与 next 两个 tag 指向不同版本](/images/dsh-rc5/fig2.png)

这不是笔误。发布流程把「预发布」和「稳定」分开了，rc 一号线一直挂在 next 上，latest 要等正式版才动。

## 四个版本的账

rc.1 到 rc.5 中间隔了三版，内容差得很多。

rc.2 是个大版本，Git Worktree 支持按分支、提交和文件类型审阅改动，SubAgent 支持具名与多层协作，桌面端可以在设置里切换正式版和开发版更新渠道，后面还跟了二十多条体验优化。

影响日常的有这么几条：向上回看聊天时自动加载更早消息并保持阅读位置、粘贴图片不再要求输入框聚焦、会话恢复或压缩后明确提示新旧上下文的替换关系、API Key 改成掩码显示且不显示完整旧密钥。最后这条我印象最深，之前那个版本会把旧密钥原样显示在界面上。

rc.3 只加了一个实验性的会话导入插件，把 DeepSeek App 或网页版的历史会话导进工作区，外加修了 Windows 应用内更新打不开的问题。

rc.4 两条修复，模型返回异常消息时的处理，以及桌面端 LiveShare 建不了分享。

rc.5 一条，就是前面那条搜索。

我这台机器一直停在 rc.1，因为 11 月 13 日试过 rc.3，有六个插件被禁用，回退了。这次升 rc.5，想看看那批插件活没活。

![rc.1 到 rc.5 中间隔了三版](/images/dsh-rc5/fig1.png)

## 第一次装，卡在一个 /tmp 路径

按 profile 的依赖改好版本号，`pnpm install` 第一轮直接失败。

报错是 `Could not install from "/tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz" as it does not exist`。

这个插件我写过。它的依赖写的是 `/tmp` 下的一个 tarball，那个目录早就被清了。

当时我以为这只是「一个插件装不回来」。这次才发现代价更大：它会让整个 profile 的依赖树解析失败，二十多个包一个都装不上。

插件本身没坏，是它的来源路径不可重现。一个用 `/tmp` 装的插件，能拖死整个 profile 的重装。

我把路径改到 `~/.dsh/plugin-archives/` 下那份备份，4,162 字节还在，pnpm 一下就过了。

这里有个细节容易忽略。pnpm 报的是「依赖树解析失败」，不是「这个包没装上」。它连别的包都不去解析了，因为整棵树的前提被证伪。所以一个来源失效的插件，代价不是它自己缺席，是所有插件一起装不上。

![两次 install 失败的对照](/images/dsh-rc5/fig3.png)

## 第二次装，卡在一个不再发版的包

路径修好，第二轮报错换了：`No matching version found for @deepseek-ai/dsh-experimental-client-ui-agent-team@0.2.2-rc.5`。

我去查了这个包在官方源上的发布记录。

末次发布是 `0.2.2-rc.1`，时间 2026 年 11 月 4 日 09:09。registry 的 `modified` 字段也停在那一刻。

换句话说，rc.2、rc.3、rc.4、rc.5 这四版发出去的时候，这个包一次都没跟着动过。

不是镜像延迟。我确认过，npmmirror 和 registry.npmjs.org 上的信息一致，它就是没发。

## 配置树对比：八进三出

我做了件笨办法的事。把 rc.1 和 rc.5 的完整配置树各 dump 一次，逐行对比。

rc.1 是 1,319 行，rc.5 是 1,355 行，多 36 行。插件 id 层面，rc.5 新出现八个：

`agent-team`、`tool-agent-team`、`ui-account-notifications`、`mcp-manager`、`office-kit`、`workspace-dependencies`、`skill-office`、`forge-panel`。

rc.1 有、rc.5 里没了的三个：`account-controller`、`office-to-pdf`、`optional-forge-panel`。

看到 `agent-team` 和 `tool-agent-team` 出现在新增列表里，我明白那个包为什么不发版了。

它被吸收进本体了。原来要靠 `dsh-experimental-client-ui-agent-team` 这个包提供的能力，现在 `@deepseek-ai/dsh-agent-team` 和 `@deepseek-ai/dsh-tool-agent-team` 直接在主包里。

其余三处也是同样的重组：`office-to-pdf` 并进了 `office-kit`，`account-controller` 换名成 `ui-account-notifications`，`optional-forge-panel` 去掉了 optional 前缀。

## 把那两个包删掉，树一点没变

我建了一个测试 profile，装 rc.5，但把这两个 agent-team 包整个从依赖和 bundle 列表里删掉。

装完 dump 配置，218 个插件 id。

再和「装着这两个包的 rc.5」比：一样是 218 个，一个不多一个不少。

删掉它们不损失任何东西，因为它们提供的东西本体已经有了。

![rc.1 与 rc.5 的配置树差异](/images/dsh-rc5/fig4.png)留着反而多一层版本约束，卡住了整次升级。

## 升级完不等于换了

本体升到 rc.5，profile 也重装好，我以为结束了。

`lsof` 一看，3080 端口上还是那个从 11 月 14 日起着的进程。它跑的是 rc.1 时代装进内存的代码。

`dsh --version` 显示 rc.5，服务本身没变。这两件事是分开的。

原因是 Node 进程启动时把模块加载进内存，之后磁盘上的包换成什么都不影响它。npm 装包改的是文件，进程改不了自己。

杀掉重启，新起的服务才真的用上 rc.5。日志里 sticky-wall 插件就绪、memorix 的 MCP 服务起来、218 个插件全加载，没有一条禁用记录。

![照踩过的顺序倒回去的四步](/images/dsh-rc5/fig5.png)

顺带说一个 npm 的细节。npm 12 在全局安装时拦了五个包的 install script，其中两个是原生模块，koffi 和 node-pty。我担心终端会坏，去翻了目录，node-pty 带着 `prebuilds/darwin-arm64/pty.node`，预编译好的，不跑 install script 也能用。

## 所以升级该怎么做

照我这次踩的顺序倒回去，四步。

第一步，别用 `npm i -g` 的默认行为，要么写 `@next`，要么写全版本号。第二步，先确认 profile 里有没有指向 `/tmp` 或临时目录的依赖，有就先修。第三步，rc.5 下那两个 agent-team 包可以整条删掉，留着只会卡版本。第四步，升完重启服务，别信 `dsh --version`。

至于 rc.5 那条搜索改动值不值得专门升，看你用不用搜索。不按次收费省的是那些模型反复触发查询的场景，如果你的会话主要是读代码改文件，这条感知不强。

但 rc.2 里那批东西，Git Worktree 按分支审阅、SubAgent 具名多层，是实打实的功能。停在 rc.1 的人，欠的不止一条搜索。

你的 dsh 现在挂在哪个版本？
