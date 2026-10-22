---
layout: post
title: "DeepSeek Harness 装插件，我撞了三堵墙"
date: 2026-10-18 08:15:28 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/dsh-plugin-index/cover.png
---

![DeepSeek Harness 装插件，我撞了三堵墙](/assets/images/dsh-plugin-index/cover.png)

## Everything is a Plugin，现在轮到「插件从哪来」

DeepSeek Harness 一直喊的那句 Everything is a Plugin，说的是功能都做成插件。0.2.1 这个版本往前挪了一步，开始管「插件从哪来」这件事：rc.1 里加了一条「允许用户从自定义插件索引源中发现和安装插件」。

我把这条链路从头跑了一遍。先读懂版本更新，再自己写一份索引挂上去，最后在真实界面里点 Install 装了一个第三方插件。中间撞了三堵墙，每一堵都值得写下来。

## 先把版本关系理清

这里有三个版本号容易混。npm 上 dsh 有三个 dist-tag：latest 停在 0.2.0-rc.2，next 是 0.2.1-rc.2，还有一个 alpha 指向 0.2.1-alpha.5。我这台机器原来装的是 0.2.1-alpha.3，正好夹在中间。

rc.1 和 rc.2 是同一天发的，前后差三小时：rc.1 是 10 月 17 日 13:22，rc.2 是 16:16。

rc.2 只有两条：适配 DeepSeek-V4.1-Flash-1017，以及适配 DeepSeek Search 与 Fetch API（在网页搜索设置里开启）。真正的重头是 rc.1，中文发布说明我数了一下，102 条变更。

跟插件分发相关的有这四条：

- 允许用户从自定义插件索引源中发现和安装插件
- 插件管理页新增「让 Agent 创建插件」入口，可保留草稿进入创造模式，发送需求后才开始执行
- 支持插件启用、停用、卸载及更新后的依赖同步；启用热重载时更新直接生效，无法热重载时提示重启
- 第三方插件的版本兼容性优先按 engines.dsh 判断，未声明时沿用原有规则

最后那条看着不起眼，它是这一轮踩坑的主角。

![三个版本号与两个候选版的内容](/assets/images/dsh-plugin-index/fig1.png)

## 索引源到底是一份什么文件

官方文档把这个能力拆成三块：一个读索引的 Host 服务、一份索引格式规范、一个 Web 端的发现页。可选组合包把它挂成一行 plugin-market。

服务的配置项很少，默认值都在 schema 和 README 里能查到：

| 配置 | 默认值 | 含义 |
|---|---|---|
| indexes | 空数组 | 索引 URL 列表，按显示顺序排 |
| fetchTimeoutMs | 15000 | 单次抓取时限，含响应体 |
| maxBytes | 134217728 | 索引响应体上限，128 MiB |
| cacheTtlMs | 600000 | 成功读取的缓存时长，10 分钟 |

有个细节值得单独说：默认不配置任何索引源时，Host 一个市场请求都不会发。也就是说这个功能在你主动加地址之前是完全安静的，不会偷偷连什么服务器。

索引本身是一份 JSON，格式版本写死在字段里（format: 1）。顶层是标题、描述、元数据（分类、标签、排序键）和插件列表；每个插件有名称、标题、描述、作者、仓库，以及版本列表；每个版本要写清楚版本号、发布时间、来源和兼容范围。

来源支持两种。npm 类型就是包名加可选的自定义 registry；git 类型要求写全 url、commit 和可选的子目录。安装时的表达式由官方函数算出来，git 版本会钉到那个 commit 上，形如 git+https://github.com/…/repo.git#7c4d0ef…。

我照着这个格式手写了一份，放了两个真实存在的第三方插件，然后用包里的校验器验。第一次直接被拦下来：

> must be an integer number of Unix seconds in [0, 10^11)

publishTimestamp 要的是 Unix 秒，我按毫秒填的。改过来之后校验通过，两个插件各自算出了安装 spec，git 那种带着完整的 commit 哈希。

![从一份 JSON 到装进 profile 的四步](/assets/images/dsh-plugin-index/fig2.png)

## 三堵墙：装插件不是一条命令的事

配置层看起来干净，真装的时候连撞三下。

第一堵是版本护栏。我先按常规装了可选组合包，pnpm 解析到的是 0.2.1-alpha.4。dsh 启动时直接把这一行禁用掉：

> disabling profile plugin row "plugin-market": Plugin @deepseek-ai/dsh-plugin-market@0.2.1-alpha.4 is incompatible with dsh 0.2.1-rc.2: requirements {"@deepseek-ai/dsh-plugin-manager":"0.2.1-alpha.4", …}

后面还给了出路：想承担风险就写版本豁免，用 dsh plugin allow-version，或者在插件管理器里点。界面上看不出任何异常，功能就是不见了。这就是「装了没反应」最标准的成因。

第二堵是新版本冷却策略。那就装精确版本 0.2.1-rc.2，结果 pnpm 输出的是一行 Packages: -8，一个包都没装，退出码还是 0。

查下去才发现是 pnpm 的 minimumReleaseAge 在拦：太新的发布要先把版本写进 pnpm-workspace.yaml 的 minimumReleaseAgeExclude 才肯装。

我用 pnpm add -w 直接跑，它自己补了九条排除记录，才把 27 个包装上。发布说明里那句「因新版本冷却策略安装较旧版本时，提供说明和精确版本安装方式」说的就是这个机制，只不过在 CLI 侧它不会大声告诉你。

第三堵是副作用。pnpm add 顺手把 profile 里八个包 prune 掉了，理由是它们只在 profile 的 bundles 名单里、不在 dependencies 里。下次启动就报：

> cannot resolve profile bundle "@deepseek-ai/dsh-experimental-agent-team-web-profile" from the dsh installation or /Users/trevorwang/.dsh/profiles/web

三个办法里只有装回来是对的。而这个包在 npm 上只有 0.1.x 线（最高 0.1.6-alpha.2），装回 0.2.1-rc.2 的运行时之后，又被第一堵墙咬了一次：ui-agent-team 这一行同样因为版本不兼容被禁用。

同一套护栏，三个小时里咬了我两次。

![三堵墙与它们的连锁反应](/assets/images/dsh-plugin-index/fig3.png)

## 真界面走一遍

配置都通了，剩下的用界面验。路径是插件页、实验性插件、更多，打开 Custom plugin markets 开关。它默认是关的，关着的时候插件页只有 Installed 一个标签。

打开之后多出 Discover，空态写得很直白：

> No plugin markets yet
> Add a plugin market index to browse and install its plugins here.

点 Add index source，弹窗里一句说明加一条警告：

> A plugin market is published as the URL of one index file. Discover browses the indexes listed here.
>
> Indexes come from third parties. Add only sources you trust: their plugins run with your permissions once installed.

我填了自己那个本地地址。第一次抓取失败，报的是 connect ECONNREFUSED，因为我把本地静态服务误杀了。点 Retry 之后正常，列表里出现我写的索引名，分类筛选是 All、Guard、Workspace，全是我在元数据里声明的；两个插件各自带着 Chinese UI 和 Experimental 标签，底下一行是版本、作者、日期。

点 Install，确认框长这样：插件名、版本 0.1.3、作者 LanFeng97、仓库地址，以及 Commit 7c4d0ef…。

索引里钉的那个 commit 原样出现在确认框里，加上一句「Plugins run on this machine with your permissions」。装完弹窗回显 git 地址和 commit，按钮从 Install 变成 Enable now。

装完之后 profile 的依赖里多了一条：

```yaml
dsh-workspace-create-dialog: github:LanFeng97/dsh-workspace-create-dialog#7c4d0ef91a0ad9d49efa71ec5a637a53e6d87b2f
```

界面还做了一件我没想到的事：它把索引源写进了 profile 自己的 patch 层，不是某个隐藏的数据库。

```yaml
- id: plugin-market
  name: "@deepseek-ai/dsh-plugin-market"
  config:
    indexes:
      - http://127.0.0.1:3489/index.json
```

也就是说这个地址是可以被别人看见、可以进版本库、可以随 profile 一起搬走的。

补一句边界：装完点 Enable now 之后，dump-config 里还没有这个插件的行，要重启进程才生效。这一步我没有接着往下验。

## 这份索引能拿来做什么

先说官方的实话。组合包文档里写得很清楚：目前没有官方插件市场，发现页只列出你自己添加的索引源。换句话说这个功能目前的价值不在「逛商店」，而在于你能自己开一个货架。

对私有插件分发来说这就够了。团队内部几个插件，写一份 JSON，扔在内部 HTTP 服务上，所有机器加同一个地址就能看到、能装，而且安装会钉到具体 commit，不是每次都拉最新。

代价也要看清楚。索引没有签名，索引里的名字和版本只是市场的声明，真正装什么由那个包自己的 manifest 决定，所以官方在弹窗里反复警告只加可信来源。不支持升级，装过的插件报 update-available 之后仍然要卸载再装。同一份列表由后写入者覆盖，两个窗口同时编辑会互相盖掉。索引还必须用最终地址配置，服务端返 301 会被拒绝。

性能上有实测数字：两万个插件的索引构建文本索引约 0.8 秒、占约 130 MiB 堆；十万个插件的 38 MiB 索引，反序列化那一步会阻塞主线程约 190 毫秒。官方自己也说，更大的市场得换服务端查询。

顺带说 rc.1 的另一条。「让 Agent 创建插件」的文案我在装好的客户端包找到了原文：createPlugin 是「让 Agent 创建插件」，描述是「进入创造模式，制作属于你的 DSH 插件」。

后面还有三句状态文案，分别是正在确认创造模式是否可用、暂时不可用请重新打开菜单重试、当前配置未提供创造模式。创造模式自己的描述是「用对话定制 DSH：让 Agent 编写插件，添加新功能或界面」。

货架搭好了，造货的工具也给了。剩下的问题是你愿不愿意让 agent 往你的 profile 里写插件。

你那边有没有内部插件需要这样分发？欢迎评论区聊聊。
