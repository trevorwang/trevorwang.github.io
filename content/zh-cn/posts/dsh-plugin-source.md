---
title: "DeepSeek Harness：我有个插件已经装不回来了"
date: 2026-11-17T08:11:07+08:00
draft: false
categories: ["公众号"]
tags: ["DeepSeek", "dsh", "Node.js"]
summary: "同一个 profile 里四个插件，四种依赖写法：版本号、钉 sha、本地 tarball、软链源码。今天重装，其中一种直接 ENOENT，而它先报的却是 corrupted。"
cover:
    image: "/images/dsh-plugin-source/cover.png"
    alt: "DeepSeek Harness：我有个插件已经装不回来了"
    relative: false
---

我顺手敲了一下 `dsh plugin --profile web list`，出来一行数字：8 packages。八个依赖，四行是我自己装的第三方插件，其余四行是官方的 bundle。看着挺整齐，直到我去翻它们的版本是怎么写的。

四种写法，四套逻辑。其中一种，我今天已经装不回来了。

## 先说我看到的那四条

profile 的 `package.json` 里，这四个插件的依赖声明分别是四种形态：一个纯版本号，一个纯版本号，一个钉了四十位 sha 的 GitHub 地址，一个指向 `/tmp` 下的本地 tarball。

![四种来源写法对照](/images/dsh-plugin-source/fig1.png)

四条里第一条是版本号 `1.8.0`，第二条也是版本号 `0.1.1`，第三条是 `github:LanFeng97/dsh-workspace-create-dialog#7c4d0ef...`，第四条是 `file:/tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz`。

官方文档里把这几种都叫「装插件」，`dsh plugin add` 后面接什么都能装进去。装的时候没区别，装完之后区别全在升级和重装这两件事上。

## 版本号是最省心的一种

`dsh-plugin-sticky-wall` 和 `dsh-plugin-ark-image-gen` 走的是普通 npm 依赖，写版本号。前者 1.8.0，后者 0.1.1。

这种的好处是可以查：根据 `npm view dsh-plugin-sticky-wall versions` 的输出，五个版本列出来，`0.0.0-stage`、`1.4.0`、`1.7.0`、`1.7.1`、`1.8.0`，`latest` 正好是 1.8.0。也就是说我手上是最新的，没落后。

它的代价是作者一改你就要跟。`latest` 往前走一步，下次 `pnpm update` 或者重装 profile，你拿到的就是新版本，没有中间档位可选。

## 钉 sha 的那种，冻结得很干净

`dsh-workspace-create-dialog` 是从 GitHub 直接装的，末尾挂着一段四十位的 sha。

这种写法把版本锁死在那一次提交上。作者之后改了什么、重构了几次、加了什么依赖，跟你都没关系，你拿到的东西每次重装都一模一样。想升级得自己去查新 sha，改 `package.json` 再装。

我顺手核了一下这个 sha 是不是当前最新的。仓库 `LanFeng97/dsh-workspace-create-dialog` 的默认分支是 `main`，末次 push 在 2026 年 10 月 13 日。

`commits?per_page=1` 回来第一条正好就是 `7c4d0ef91a0ad9d49efa71ec5a637a53e6d87b2f`。也就是说我钉的这一版，此刻就是 HEAD。

这个仓库目前 1 个 star。钉 sha 这种装法，在冷门仓库上其实是最靠谱的选择，因为它不依赖作者打 tag、不走 npm 发布、也不怕仓库被改名，代价只是你得自己记得来核一次。

![npm 版本列表与 GitHub 当前 HEAD](/images/dsh-plugin-source/fig2.png)

## 本地 tarball 那条，今天已经断了

`dsh-plugin-profile-ping` 写的是 `file:/tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz`。

这是我当初自己打包塞进去的。我建了个最小的 `package.json`，只放这一条依赖，跑 `npm install`，得到的是这个：

> npm warn tarball data for dsh-plugin-profile-ping@file:/tmp/dsh-plugin-build/...tgz (null) seems to be corrupted. Trying again.
> npm error code ENOENT
> npm error path /tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz
> npm error enoent ENOENT: no such file or directory

![最小复现：npm install 的原始报错](/images/dsh-plugin-source/fig3.png)

`/tmp/dsh-plugin-build` 这个目录，已经不在了。macOS 会定期清 `/tmp`，这个过程不需要你同意。

最误导人的是它先报了两行 `corrupted`，第三行才说实话 `ENOENT`。看到 corrupted，人的第一反应是包的格式坏了、去重新打包；实际原因简单得多，是那个文件本身没了。

而 `pnpm-lock.yaml` 里记的这段也一样跟着死：`version: file:../../../../../tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz`。五个 `..` 层层退到根目录，再钻进 `/tmp`。

也就是说，我这个 profile 现在只要重装一次，就会卡在这一个包上。不是整个 profile 全废，是它装不完整。

## 它其实留了退路，只是没指过去

`~/.dsh/plugin-archives/` 里躺着 `dsh-plugin-profile-ping-0.1.0.tgz`，4,162 字节，日期是 11 月 4 日。

我把依赖路径改成指向这份，再跑一次安装，结果是 `added 1 package in 102ms`。

同一个包、同一份字节，换个路径就活了。那份 tarball 的日期是 2026 年 11 月 4 日。所以我这四条里唯一坏掉的那条，修法只有一行：把 `/tmp/dsh-plugin-build/` 换成 `~/.dsh/plugin-archives/`。

顺手把它拆开看了一眼，一共七个条目。`index.js` 6,954 字节是主体，`README.md` 1,788 字节，`icon.svg` 647 字节，`package.json` 526 字节，`locale` 下两个 json 分别 199 和 196 字节，`cordis.patch.yml` 只有 71 字节。

七个条目解压后合计 10,381 字节，压成 4,162 字节的 tarball，压掉了 59.9%。

![tarball 内部的七个条目](/images/dsh-plugin-source/fig4.png)

`package.json` 里有一行 `"private": true`。带这个标记的包永远不会发到 npm 上去，所以我这份本地包没有公开版本可查，除了我自己手上的文件，没有任何备份。

## 第四种是 link 到源码目录

superagent 那个 profile 里又是另一套写法：`dsh-superagent: link:/Users/trevorwang/dsh-superagent`。

`link:` 是软链到本地源码目录，不是拷贝。改完代码不用重新安装，下次启动就是新的。开发插件的时候这是最顺手的形态，试错成本几乎为零。

代价也直接：没有版本号，没有 sha，没有 tarball。目录一挪、名字一改，这个 profile 就变成了指向空气的软链。它能不能装回来，取决于你那个源码目录还在不在。

## 每个插件都自带一张插条

四种来源写进 `package.json` 的方式差别很大，但装进去之后长得一模一样，因为它们都遵循同一个约定。

四个插件的 `package.json` 里都有一行 `dsh.bundle.patch`，指向自己目录下的 `cordis.patch.yml`。那个文件里是一条插入声明，写明自己的 loader id 和包名。

`dsh-workspace-create-dialog` 那份的注释写得最清楚，原文写道：「bundle patch：把本插件的 loader 行插进 web profile 名单。由 `package.json` 的 `dsh.bundle.patch` 声明，安装时作为 profile bundle 层生效。」

带前端界面的还多一层。sticky-wall 写了 `client.platform: "web"`，workspace-create-dialog 写得更细，`client.inject` 里挂了三个 UI 包：renderer、layout 和 workspace。剩下两个纯后端插件，`dsh` 字段里只有 `bundle`，没有 `client`。

所以「来源」只决定版本怎么解析，装进去之后大家走的是同一条路。这也意味着一个本地 tarball 插件和一个 npm 正式版插件，在运行时的待遇完全一样，看不出谁是野的。

## 四个里只有一个声明了版本范围

这是我觉得最该被注意的一处。

`dsh-workspace-create-dialog` 的 `dsh` 字段里除了 bundle 和 client，还有一条 `engines`：`>=0.2.0-rc.2 <0.3.0 || >=0.3.0-0`。它明确告诉你，这个插件在哪些 Harness 版本上能跑。

剩下三个，一个都没写。sticky-wall 的 `dsh` 字段就两样，`bundle` 和 `client.platform`。ark-image-gen 和 profile-ping 只有 `bundle`。

![四个插件的 dsh 字段对照](/images/dsh-plugin-source/fig5.png)

不写 `engines` 的插件，装到不兼容的版本上不会被拦。它不会在安装时报错，也不会在启动时警告你，通常是跑起来之后某个功能悄悄不动了，你得自己往回找。

这也解释了一件事：为什么升级 Harness 之后，出问题的往往是那些不起眼的小插件。声明了范围的至少还能在装的时候拦你一道，没声明的只能靠你记得它存在。

## 装得进去，不代表装得回来

我现在看一个插件，会先看它依赖那一行是怎么写的。

版本号的最省心，但也最容易被作者推着走。钉 sha 的最稳，代价是你得自己盯着上游。本地 tarball 最自由，但只要那个路径不在你的掌控里，它随时会变成今天早上我遇到的样子。link 到源码最方便开发，连版本这个概念都没有。

四种装法都能装进去，装的过程中你感觉不到任何差别。真正决定它活多久的，是那个路径在哪、谁能删掉它，以及到期之前有没有人替你留一份。

再补一句我这半天的判断：`dsh plugin` 这个命令本身不做什么，帮助里写得很直白，它只是把剩下的参数原样转发给 pnpm，在 profile 目录里执行。

所以插件的来源规则就是 npm 的来源规则，`file:` 指向不存在的文件会 `ENOENT`，跟 dsh 一点关系都没有。知道这一层之后，那些看起来发生在 Harness 身上的问题，都能直接套 npm 的常识去查。

顺便说，如果你的 `/tmp` 里也躺着某个 `file:` 依赖，不妨现在就去 `ls` 一眼。真没了的话，`~/.dsh/plugin-archives/` 里可能还留着一份。

你现在装的插件里，有几种是靠本地路径活着的？手边这几个我打算都挪进 archive 里，你觉得还有别的地方该一起清一遍吗。
