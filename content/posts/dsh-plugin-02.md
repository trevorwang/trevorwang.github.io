---
title: "DeepSeek Harness 0.2：写插件，动嘴就行"
date: 2026-09-29T08:16:34+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "9 月 28 日 DeepSeek Harness 发布 v0.2.0-rc.1。release notes 里两条和插件有关：改善插件管理界面与安装引导，创造模式完善插件开发指引并提供体验技能。"
cover:
    image: "/images/dsh-plugin-02/cover.png"
    alt: "DeepSeek Harness 0.2：写插件，动嘴就行"
    relative: false
---

## 0.2 系列的第一个候选版

9 月 28 日 DeepSeek Harness 发布 v0.2.0-rc.1。release notes 里两条和插件有关：改善插件管理界面与安装引导，创造模式完善插件开发指引并提供体验技能。

注意它在 npm 的 `next` 标签上，`latest` 还停在 0.1.7-rc.2，默认那条命令装不到。

我把新版装在隔离目录里跑（独立的 prefix 与 `DSH_HOME`），没碰本机正在跑的实例，下面的输出都来自 0.2.0-rc.1。

## 现在有三条路可走

新版把「写插件」摊成了三种入口，门槛依次降低。

第一条是 Web 侧边栏的「插件」页。启停插件条目、勾选组合包、安装或删除外部组合包，都在界面上点完，不用手改配置文件。

第二条是创造模式。新建任务时选它，用一句话让 agent 去装。官方给的示例提示词就是这个口气：

> 帮我写一个 DSH 插件，在侧栏增加「项目笔记」入口，列出当前工作区的 Markdown 文件，点击后能预览内容。完成安装并验证页面能打开。

第三条还是手写三件套，最短也最可控。三条路通往同一套机制，所以先把机制看明白，前两条才不是黑盒。

## 最短路径仍然是三个文件

官方教程给的目录就三个文件：

```text
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 这个包贡献的配置层
└── index.js           # patch 行引用的插件入口
```

`package.json` 里真正要紧的是 `dsh.bundle` 这一段：

```json
{
  "name": "dsh-hello-plugin",
  "type": "module",
  "main": "index.js",
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`index.js` 是 Cordis 插件的入口，加载时框架用一个上下文调用 `apply`，插件通过这个 `ctx` 注册自己贡献的一切：

```js
export const name = 'hello-plugin'

export function apply() {
  console.log('[hello-plugin] plugin loaded!')
}
```

`cordis.patch.yml` 就是你贡献的那一行。这里写包名而不是文件路径，Node 才能从 profile 的 `node_modules` 里找到你：

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

装它是一条命令。命令行入口叫 `dsh`，`dsh plugin --profile <name>` 会把参数转发给 profile 目录里的 pnpm：

```sh
dsh plugin --profile demo add ./hello-plugin
```

我在这里踩了个坑：我 `cd` 进包目录里又写 `add ./hello-plugin`，DeepSeek Harness 按当前目录解析，依赖记成了 `link:/private/tmp/hello-plugin/hello-plugin`，多一层路径，启动直接报 `cannot resolve profile bundle`。这个坑两个版本上我都复现了，必须在包的父目录执行。

## 装完先看两层东西

第一层是配置：`dsh --profile demo --dump-config` 打印组合后的完整树，我只截层标记，第二个就是我加的。

```
# == @deepseek-ai/dsh-base
# == dsh-hello-plugin
```

第二层是运行：启动后屏幕上出现 `[hello-plugin] plugin loaded!`。到这里，从建目录到看见这行，三条命令。

## 新版给插件开发者加的两样东西

一样是命令行。新版多了一个 `--dump-config-schema`，打印 profile 条目和 patch 的 JSON Schema。我这份输出 15.6 万字符、73 个定义，`entry`、`patch`、`patchList`、`includePatch` 都在里面。以前想确认「这一行到底能写哪些字段」只能翻源码，现在可以先问 schema。

另一样在应用里。创造模式自带一份插件开发指引，原文是这么写的：

> 创造模式具备标准任务工具，并增加运行时检查、持久化插件管理，以及 Cordis 插件和 Agent 预设的开发指引。可以编写插件来添加功能或界面，也可以组合工具和提示词，创建适合特定任务的模式。

指引里还专门解释了插件和模式的关系：插件给 DeepSeek Harness 加能力，比如工具、服务连接或界面入口；模式是一份 Agent 预设，决定这次任务能用哪些工具、按什么方式干活。自定义模式里也能用自己写的插件。

「怎样让成果生效」那节写得更像提醒：可以让 Agent 完成安装并验证实际效果，插件可能即时加载也可能要重启，取决于改了什么。

## 让 agent 动手，边界写在哪

创造模式里能用的那个工具叫 `plugin_manager`。我从仓库里一份录制快照中取到了它的真实描述：

> List plugins or bundles in the current profile, enable or disable them, install a bundle, or remove an installed bundle. Every action requires danger-full-access permission or approval for this call. Approval does not change the session permission mode. Changes affect every session in this profile.

它接受九个参数，除了 `action` 和 `target`，还有 `registry`（指定去哪个源找）、`approvedBuilds`（代你批准依赖的构建脚本），以及和版本豁免有关的一对 `runtimeVersion`、`acceptRisk`。

这段描述里有三条边界值得单独拎出来。

第一，每个操作都要 `danger-full-access` 权限，或者这一次调用单独批准；批准不改变会话的权限模式，也就是说批准一次不等于长期放行。

第二，改动写进 profile，影响这个 profile 下的所有会话。不是只影响你当前这个对话。

第三，装进来的 Host 代码在宿主进程里跑，不受工作区沙箱限制。这句话的分量在于：插件和它能碰到的权限，等于你自己在这个终端里的权限。

## 装不兼容的插件会被拦下来

新版在包管理上加了一道前置检查：你的插件声明的运行时版本和当前这版 DeepSeek Harness 对不上，操作会在 pnpm 跑起来之前失败，所以什么都不会下载、构建脚本也不会执行。

真要装，就得走版本豁免：豁免写进 profile 自己的 `compatibility.json`，把精确的 `包名@版本` 映射到精确的运行时版本，而且必须带 `acceptRisk: true`。CLI 上对应三条命令：

```sh
dsh plugin --profile web version-exemptions
dsh plugin --profile web allow-version <包名@版本> --dsh-version <运行时> --accept-risk
dsh plugin --profile web revoke-version <包名@版本> --dsh-version <运行时>
```

插件升级和 DeepSeek Harness 升级都不继承授权，得重新决定。这个设计我觉得是对的：豁免是一次性的知情决定，不是一张长期通行证。

## 层的顺序，和一条最容易吃亏的规则

生效配置在空根之上按四层组合，后面的层按行压住前面的：

1. profile 的 `dsh.profile.bundles` 列表里的各个组合包 patch，按列表顺序
2. profile 自己的 `cordis.patch.yml`
3. home 级的 `$DSH_HOME/cordis.patch.yml`，各 profile 共用
4. 命令行上的每个 `--patch <path>`，按 argv 顺序

规则里有一句最容易吃亏：patch 替换目标行的整个 `config`，不是深合并各个键。想覆盖前面某一层的那一行，必须把那一行需要的每个键重述一遍，只写你改的那个会把别的键清掉。

反过来也是好消息：用户能在自己 profile 的 `cordis.patch.yml` 里覆盖你那一行，不用改你的包。所以默认值尽量给成用户大概率会留下的那种。

## 同一场失败，我在两个版本上各跑了一次

官方 Cordis 教程说插件加载失败会明确报错，但模块解析不到（路径或包名拼错）只记日志、不崩。

我在 0.1.6-alpha.2 上试的时候和这句对不上：让 `apply` 抛异常，整棵插件树加载失败、进程 exit 1，第一行就点名了插件条目；包名写错，同样 exit 1，报 `Cannot find package`。

换成 0.2.0-rc.1 再试，两种失败都变成警告，进程继续活着：

```
dsh: warning: 1 entry did not activate
hello (dsh-hello-plugin): Error: apply exploded
```

包名写错那条更短，只一行 `hello (dsh-hello-pluginX): failed to import`。

这是新版一个真实的行为变化，我的判断是它更友好了，但也更安静：插件没生效不一定是因为它没装上，可能只是那一条 entry 没激活。升级之后如果你发现「装了没反应」，先去启动输出里搜 `did not activate`，比反复重装快得多。

## 最后

想动手的最小路径还是三步：建目录放三个文件，`dsh plugin --profile <你的 profile> add ./你的插件目录` 装上，`--dump-config` 确认层出现了。想省事就直接在创造模式里开口，让它装完再验证给你看。

我下一步想试的是 `--dump-config-schema`，用 schema 生成一份「插件行可用字段」的速查表，比翻文档省事。

你打算给 DeepSeek Harness 挂什么？给模型加个工具，还是像社区里那批插件一样往侧边栏塞张卡片？评论区聊聊，也许下一篇就写它。

参考资料：
- 打包与安装插件（官方）：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/user/develop/basic/publish.zh.md
- 通过提示词配置持久化插件（创造模式）：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/user/develop/practice/dynamic-cordis.zh.md
- Plugin Manager 包参考：https://github.com/deepseek-ai/deepseek-harness/blob/main/packages/boot/plugin-manager/README.zh.md
- DeepSeek Harness v0.2.0-rc.1：https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.2.0-rc.1

验证边界：三件套的安装、`--dump-config` 层输出、启动日志、两个失败行为、`--dump-config-schema` 的输出规模，以及 `plugin_manager` 的参数与描述，都在本机 0.2.0-rc.1 上核对过（装在隔离 prefix 与隔离 `DSH_HOME`，没动正在运行的实例）；0.1.6-alpha.2 的对照数据也来自本机实测。创造模式里让 agent 装插件这条路径没有真跑，因为本机没有可用的模型 key，那段依据的是官方文档与应用内指引原文。
