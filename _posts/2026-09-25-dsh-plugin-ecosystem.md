---
layout: post
title: "6 周 4287 个插件：DeepSeek Harness 的生态是怎么管住自己的"
date: 2026-09-25 09:46:36 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/dsh-plugin-ecosystem/cover.png
---

![6 周 4287 个插件：DeepSeek Harness 的生态是怎么管住自己的](/assets/images/dsh-plugin-ecosystem/cover.png)

## 打开这个列表，第一眼是 4287

`awesome-dsh-plugin.com` 顶上有个搜索框，右下角写着 `4287 / 4287`。下面一排分类标签：UI 731、工具与能力 567、开发与运行时 312、会话与消息 281、工作流 260、用量与计费 226、模型 206、记忆 202…

![Awesome DSH Plugin 首页，收录 4287 个插件](/assets/images/dsh-plugin-ecosystem/01-site-home.png)

这个数字不是慢慢攒的。仓库 8 月 13 日建，到 9 月 25 日刚满六周，提交七千多次，单周峰值 2178。⭐16.8k，fork 3.3k，收录还在按小时往前推。

生态长得快不算新闻。我真正想弄清楚的是另一件事：这么多第三方插件，你怎么知道哪个能装。

## 一个插件一个文件，是被合并冲突逼出来的

先说列表本身怎么维护。它的两个 README 和那个站点**都是生成的**，源数据在 `data/plugins/`，一个插件一个 YAML 文件：

```yaml
url: https://github.com/owner/repo
name: owner/repo
category: ui
description:
  en: One-line description ending with a period.
  zh: 一句话描述，以句号结尾。
```

contributing.md 里专门写了一句为什么这么做：「以前大家都往同一个 README 段落的同一处追加，合并一个 PR 就会把下一个弄坏。」

这句比很多架构文档都实在。四千多个条目，如果都往同一个文件的同一处插，PR 队列会自己把自己堵死。拆成一个文件一个条目之后，每份提交落在不同路径，冲突面就消失了。

代价是生成器得管得细，它确实管得细。我在后面会真的踩它一次。

## 可安装的单位不是「插件」，是「组合包」

这块读官方文档时我卡了一下，因为它把两件事分得很清楚：

- 组合包是个 npm 包，`package.json` 里声明 `dsh.bundle`，回答的是「这个包贡献什么」，内容是一个 patch 文件
- profile 是 `$DSH_HOME/profiles/<name>` 下的目录，声明 `dsh.profile`，回答的是「这套配置由哪些组合包、按什么顺序组成」

组合包是你写出来分发的东西，profile 是用户用 `dsh --profile <name>` 启动的东西。没有东西同时是两者。

我打开自己机器上的 profile 看了一眼，结构很直白：`package.json` 里列着四个官方组合包，旁边是 `cordis.yml` 和 `cordis.patch.yml`，两个文件里都只有一行 `[]`，但注释把层次交代清楚了：

> dsh profile root — an empty entry list. The tree is composed as patches: each bundle in package.json's dsh.profile.bundles, then cordis.patch.yml, then any --patch overlays. Edit cordis.patch.yml, not this file.

组合顺序就是：bundles 按序铺开 → 这个 profile 自己的 patch → `--patch` 覆盖层。

还有个细节我挺喜欢：`dsh plugin` 是把参数透传给 pnpm 的。我跑 `dsh plugin --profile web --help`，出来的是 pnpm 的帮助页。也就是说 profile 本质上是个 pnpm 工作区，**插件就是它的依赖**，装插件走的就是 `pnpm add` 那一套。

## 每个插件会碰什么，被扫成了公开数据

这才是我认为整个生态最值得看的地方。

站点上每个插件都能查到它会碰什么，数据在 `data/capabilities.json`，扫它的是一个叫 `dsh-trust-check` 的工具。目前扫过 3845 个，分布是这样：

- 读文件 75.0%
- 写文件 58.8%
- 读环境变量 57.4%
- 联网 56.8%
- 执行 shell 42.5%
- 碰凭据 11.9%
- 调模型 9.1%、动态执行代码 9.0%、访问宿主运行时 7.7%

单项能力不可怕，组合才可怕。同时要 **凭据 + shell + 网络 + 写文件** 这四样的，有 178 个。

扫出来会标红线，一共 **454 个** 插件被标了。红线大致三类：

- 读凭据并且有网络访问，这是最常见的一类
- 用明文 `http://` 访问某个域名
- 用字面 IP 访问网络，我看到有插件写的是 `169.254.169.254`，那是云上的元数据端点

得说明白：**这是能力披露，不是判决。** 标红只说明它有能力做某件事，不代表它做了。我翻到的几条明显是示例仓库的占位值，有个插件的红线里写着 `evil.com:3080` 和 `1.2.3.4`。站点自己的免责声明也写得很清楚，收录不等于背书。

但把能力扫成公开数据这个动作本身很有用：你不需要读完四千份源码，也能先筛掉一大批。

## 客户端从哪来：官方只给了两条路

翻官方站点的时候我发现一件事：**它没有「客户端下载」这个入口。** 中文页整页搜不到「下载」「桌面」「客户端」任何一个词。

官方给的是两条路（[deepseek.com/harness](https://www.deepseek.com/harness/) 的「开始使用」）：

```sh
# 快速体验：装好 Node.js，用 npx 起 Web UI
npx @deepseek-ai/dsh web

# 源码安装
git clone https://github.com/deepseek-ai/deepseek-harness
```

跟着官方命令走会踩一个坑：npm 上这个包的 `latest` 还停在 `0.1.5-rc.3`，而 `next` 已经是 `0.1.7-rc.2`。所以直接敲 `npx @deepseek-ai/dsh web` 装到的是旧版。要新版得显式指定：

```sh
npx -y @deepseek-ai/dsh@next web
```

## 官方桌面端在发，只是你搜不到

**这一节是更正。** 我第一版写的是"官方桌面端没有公开下载"，那是错的。

官方自己的桌面端在仓库的 `apps/desktop`，自带 Python 和 pnpm 整套运行时。它确实在分发，只是不放在下载页上——它是**更新通道**的一部分，清单在两份 YAML 里：

```
https://download.deepseek.com/dsh-desk/feeds/mac-arm64/nightly-mac.yml
https://download.deepseek.com/dsh-desk/feeds/win-x64/nightly.yml
```

包在 `bin/mac-arm64/` 和 `bin/win-x64/` 下，命名规则是 `deepseek-harness-<版本>-<平台>.zip` 或 `.exe`。mac-arm64 那个 355MB，Windows x64 是 274MB。只有这两个平台，只有 nightly 一条通道（`latest` 那份不存在，说明还没有稳定版），而且是从 `0.1.7-rc.1` 才开始发的。

我上次判断错，是因为**拿"下载"这两个字去搜，搜完就下了结论**。`@deepseek-ai/dsh-desktop` 在 npm 上确实是 404（那个私有 Host 包本来就不发布），官网也确实没有下载入口——但包一直在发。这块我单独写了一篇，里面有清单原文、sha512 校验和包内容拆解。

## 社区也有好几个壳子

社区那几个桌面端仍然值得知道，因为官方只发 macOS arm64 和 Windows x64，而且还在 nightly。它们也是你去官方仓库 Discussions（7673 条讨论）里搜 desktop 最容易撞见的东西：

- `dataelement/dsh-desktop`，⭐9,132，MIT，官网 `dshdesktop.com`。mac arm64/x64 的 DMG 加 Windows 的 setup.exe（162MB），macOS 已签名并公证。发版很勤，9 月 12 到 23 号发了六版，Windows 安装包累计下载一万三千多次
- `zsyu9779/dsh-desktop`，用 Wails 做的，包很小：darwin-universal 的 DMG 只有 9MB，exe 11MB，Linux 4MB
- `fufankeji/deepseek-harness-studio`，⭐661，GitHub 上没有 release，下载放在它自己那边
- Microsoft Store 上还有一个「DS桌面版」，宣传语直接写「无需 Node.js 和命令行」

这些都不是 DeepSeek 官方出品，README 里也写明是第三方在做。挑的时候有个地方要留神：壳子跟的是某一个 dsh 版本。比如 dshdesktop 现在建在 `0.1.5-rc.2` 上，而官方已经到 `0.1.7-rc.2`，中间隔着语音输入、账号登录这些新东西。生态还在动，壳子未必跟得上。

找的时候小心一点：这些壳子本质上是把你本机的 dsh 包了一层，它拿到的权限就是 dsh 的权限。官方的能力扫描只覆盖插件，不覆盖桌面壳。

官方的几个入口，一起放这儿：

- 中文站点：https://www.deepseek.com/harness/
- 开发者文档：https://deepseek-harness.github.io/deepseek-harness/guide/quickstart
- 官方桌面端 mac 更新源：https://download.deepseek.com/dsh-desk/feeds/mac-arm64/nightly-mac.yml
- 官方桌面端 win 更新源：https://download.deepseek.com/dsh-desk/feeds/win-x64/nightly.yml
- 社区插件（就是官方站点上那个「社区插件」链接）：https://github.com/topics/dsh-plugin
- 社区桌面端的集散地（官方仓库 Discussions）：https://github.com/deepseek-ai/deepseek-harness/discussions
- 源码：https://github.com/deepseek-ai/deepseek-harness

## 装的时候还有一道闸

安装命令站点直接给两行：

这两行在站点上直接给，右边还有一键复制：

![插件详情页的安装区，以及下面的授权提醒](/assets/images/dsh-plugin-ecosystem/02-detail-install.png)

```sh
# 从 npm（预编译）
dsh plugin --profile web add @deepseek-harness-tui/dsh-tui

# 从 GitHub（首次会问 allowBuilds，按提示重试）
dsh plugin --profile web add github:owner/repo
```

从 GitHub 装的那条，第一次跑多半会停在 `ERR_PNPM_GIT_DEP_PREPARE_NOT_ALLOWED` 或 `ERR_PNPM_IGNORED_BUILDS`，因为 pnpm 默认不让第三方包跑安装脚本。dsh 会告诉你该往 profile 的 `pnpm-workspace.yaml` 里 `allowBuilds` 加哪个键，重跑就过。

这一步的措辞我照抄一下，它说得比很多安全文档都直：

> 你装的任何插件都在用你自己的权限跑第三方代码，它能读你的文件、用你的凭据、访问网络，工具审批并不隔离它。允许一次构建是一个信任决定：只装你信任的来源，并且钉住提交（`github:owner/repo#sha`）。

**我没有真的装。** 停在这一步了，因为按上面那句话，装一个第三方插件就是在自己机器上跑别人的代码。这篇文章不需要这个证据。

## 我照着文档提交了一个插件，然后踩了坑

收录要求里有一条挺狠：**只声明 `dsh.client`** 是最常见的被拒原因，那样根本装不上，必须声明 `dsh.bundle`。另外仓库要创建满 1 天（自动检查，没有提交数门槛，他们写了句「提交历史衡量的是开发习惯，不是质量」），要有真代码，要打 `dsh-plugin` topic，而且**描述会被拿去对代码**。

那就从最省事的部分试起：加一个条目。

我在本地 clone 里建了个文件，把文档给的模板填上，跑生成器，被拒了：

```
A description containing ": " must be quoted:
  en: Demo entry: verifies the one-file submission flow end to end.
becomes
  en: 'Demo entry: verifies the one-file submission flow end to end.'
```

YAML 把 `: `（冒号加空格）当成了嵌套键。这条坑 contributing.md 里明明写着，我还是踩了。好笑的是一看到报错就懂了，因为它顺手把改法给了出来。

加引号重跑：

```
README.md: regenerated (4288 entries)
README.zh.md: regenerated (4288 entries)
```

`git status` 里除了我那个新文件，只有两个 README 被改动，各多一行：

```
README.md:908:   - [hermes-test/dsh-demo](...) - Demo entry: verifies the one-file submission flow end to end.
README.zh.md:908: - [hermes-test/dsh-demo](...) — 演示条目：验证一个文件即完整提交的流程。
```

**一个 YAML 文件，就是全部提交。** 中文那份也是它自己长出来的，我只填了 `zh` 字段。

## 所以我怎么挑插件

把上面几条拼起来，我自己的顺序是这样：

1. 看 30 天下载量，排在 star 前面。 这些插件的 star 经常是被宿主仓库带起来的，数据里 star 榜首那个 29,567 是仓库自己的，插件只是它 `examples/` 下的一个目录。下载量更接近「有多少人真在用」。
2. 看收录时间。 详情页有 `Added` 字段，8 月 14 日就进来的，至少走过了六周里的迭代。
3. 看能力清单。 我一般不装同时要凭据、shell 和网络这几样的。
4. 从 GitHub 装就钉住 commit，站点自己也是这么建议的。

至于「它到底做了什么」，那还得读代码。能力扫描的作用是把范围从四千多缩到手数得过来的几个，剩下那几步没有捷径。

## 小结

六周长到 4287 个插件，这个速度说明 DeepSeek Harness 那套「一切都是插件」的设计确实能被别人接住。但让它不像一个野生集市的，是三个小决定：一个条目一个文件，为了能合并；能力扫成公开数据，为了能筛；安装期的构建审批，为了让「你在授权什么」这件事至少是显式的。

链接：

- 站点：https://awesome-dsh-plugin.com
- 列表仓库：https://github.com/awesome-dsh-plugin/awesome-dsh-plugin
- 官方《打包与安装插件》：https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/user/develop/basic/publish.zh.md
- 另一个入口：`dsh-market`，能在这个列表里直接浏览和安装，安装数 82,491，全列表第一
