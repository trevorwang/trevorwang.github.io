---
title: "dsh 0.2.2-rc.1：这次发布改了 0 行代码"
date: 2026-11-04T22:40:00+08:00
draft: false
categories: ["公众号"]
tags: ['DeepSeek', 'dsh', 'AI Agent']
summary: "拆包逐文件比 md5：8 个包 290 个文件，改动只有 8 处，全在 package.json。版本号从 alpha 走到 rc，代码一行没动，但依赖全线升级，插件得跟着抬版本。"
cover:
    image: "/images/dsh-022rc/cover.png"
    alt: "dsh 0.2.2-rc.1：这次发布改了 0 行代码"
    relative: false
---

今天下午五点二十，DeepSeek Harness 发了 0.2.2-rc.1。

从 alpha 转 rc，听着像是个里程碑。我把两个版本都拆开比了一遍，结论是：一个字节都没改。

## 先说结论

`0.2.2-rc.1` 发布于北京时间 2026 年 11 月 4 日 17:20（`2026-11-04T09:20:32Z`）。

我拿它和 2026 年 11 月 3 日发布的 `0.2.2-alpha.3` 做了逐文件 md5 比对，范围是顶层包、六个核心子包、以及底层框架 `cordis`，一共 290 个文件对齐算了一遍。

结果：每个包里唯一变的文件，都是 `package.json`。

`dist`、`src`、`locale`、`README` 全部一模一样。这不是「改动很小」，这是零改动。

## 我怎么比的

命令行就三步，没用什么特殊工具：

`npm pack @deepseek-ai/dsh@0.2.2-alpha.3` 和 `@0.2.2-rc.1` 各拉一份 tarball，解包到两个目录，然后按相对路径逐文件算 md5。

顶层包 20 个文件，六个子包各 30 到 60 个文件，`cordis` 32 个文件。两边同名文件全部对齐算一遍，只记下哈希不一致的。

| 包 | 文件数 | 内容变化的文件 |
|---|---|---|
| dsh（顶层） | 20 | 1 个，package.json |
| dsh-agent-preset | 48 | 1 个，package.json |
| dsh-plugin-manager | 60 | 1 个，package.json |
| dsh-web-app | 32 | 1 个，package.json |
| dsh-client-ui-cordis | 30 | 1 个，package.json |
| dsh-tool-fs | 34 | 1 个，package.json |
| dsh-tool-cordis | 34 | 1 个，package.json |
| cordis | 32 | 1 个，package.json |

没有新增，没有删除。八个包八处改动，全是版本号和依赖范围那一行。

![](/images/dsh-022rc/fig1.png)

## 唯一真正变了的东西

代码没变。但依赖版本号全换了一茬，这是这次发布唯一的实质动作：

| 包 | alpha.3 | rc.1 |
|---|---|---|
| `@deepseek-ai/cordis` | 4.0.6-alpha.2 | 4.1.0 |
| `cordis-plugin-loader` | 1.0.7-alpha.2 | 1.1.0 |
| `cordis-plugin-include` | 1.0.11-alpha.2 | 1.1.0 |
| `cordis-plugin-timer` | 1.1.8-alpha.2 | 1.2.0 |
| `@deepseek-ai/schemastery` | 3.18.6-alpha.2 | 3.19.0 |
| `@deepseek-ai/cosmokit` | 1.8.7-alpha.2 | 1.9.0 |

一句话总结：把整个 cordis 家族的 alpha 后缀去掉，抬成正式版本号。

但这里有个细节值得说。我把 `cordis` 4.0.6-alpha.2 和 4.1.0 也拆开比了，它的 32 个文件里同样只有 `package.json` 不同。也就是说连框架层都是原样搬过去的，只换了门牌号。这个比上面那张表更有意思。

那次「版本号从 alpha 变正式」，改动量是零。

![](/images/dsh-022rc/fig2.png)

## 所以这次发布到底图什么

图一个 tag。

发布之前，`latest` 指向 `0.2.1-rc.5`，`next` 指向 `0.2.2-alpha.3`。想装最新的得手动写全版本号。

发布之后，`latest` 和 `next` 都指向 `0.2.2-rc.1`。这意味着 `npm i -g @deepseek-ai/dsh` 不加版本号，默认装的就是它。

对大项目来说这不是小事。alpha 线是「我们自己敢用」的标记，rc 是「你也来试试，接口已经定死了」的标记。改功能是开发，改标记是承诺。这一版做的是后者。

## 升级花了不到一分钟

全局装一条命令，`npm i -g @deepseek-ai/dsh@0.2.2-rc.1`，跑完 `dsh --version` 就是 `0.2.2-rc.1`。

第二条命令不是升级，是修 profile。这条很多人会漏，我上次就漏了：

`dsh plugin --profile web add @deepseek-ai/dsh-cua-bundle@0.2.2-rc.1 @deepseek-ai/dsh-experimental-plugin-market-bundle@0.2.2-rc.1 @deepseek-ai/dsh-experimental-client-ui-agent-team@0.2.2-rc.1`

pnpm 报回来：解析 62 个包，复用 41 个，下载 21 个，新增 22 个，耗时 939 毫秒。

装完我先用 headless 模式冒烟了一次，让它「只回答两个字：收到」。它回了「收到」，说明 profile 能正常启动、模型路由通、没有插件卡在 disable 状态。这一步比看界面快。

为什么要重跑？因为 dsh 的 profile 是一层叠一层的插件堆，全局升了 dsh 之后，profile 里那些还钉在 alpha.3 的插件和主程序版本对不上，启动时会被版本校验直接 disable。

上次升 alpha.3 我只升了全局，结果六行插件被关掉，`storage-domain` 就在其中。它一挂，工作区、会话列表、定时任务跟着全卡死，网页版直接废掉，最后靠这条命令才救回来。

所以我的顺序固定了：先全局，再 profile，最后验证。少一步就是给自己找麻烦。

![](/images/dsh-022rc/fig3.png)

## 上次没救回来的那个，这次回来了

上次升级唯一遗留的伤是 `ui-agent-team`：它版本线特殊，npm 上找不到能和 alpha.3 对齐的包，只能一直禁用着。

这次它有对应版本了。装完之后 `profile_ping` 报的插件清单里，这三个都在：

`@deepseek-ai/dsh-experimental-agent-team`

`@deepseek-ai/dsh-experimental-client-ui-agent-team`

`@deepseek-ai/dsh-experimental-tool-agent-team`

上一版被禁的 `dsh-storage-domain` 也在列表里。插件页面上，「Agent teams」那个开关是打开的，新会话的标题栏上还多了一个 Agent team 的标识。

装的时候出了一句提示，我觉得值得原样记下来：`@deepseek-ai/dsh-experimental-client-ui-agent-team` 没有声明 `dsh.bundle`，所以它是作为普通依赖装的，不是一个 profile 层。

换句话说它能用，但它在体系里的位置和那些 bundle 不一样。这种话厂商不会写在发布说明里，只在安装输出里闪一句。

![](/images/dsh-022rc/fig4.png)

## profile 里不全是官方包

顺手看了一眼 profile 的依赖清单，六条：

三个 bundle 现在是 rc.1，`agent-team-web-profile` 还停在 `0.1.6-alpha.2`，剩下两条是我自己装的，一个从本地 tgz 装，一个直接从 GitHub 仓库装。

这说明 dsh 的 profile 不是一个封闭的官方套装，是你自己拼的组合。好处是你想加什么加什么，代价是升级时得自己盯：哪些是官方的、哪些钉在某个版本、哪些根本不在官方的版本线上。

`agent-team-web-profile` 就是例子。它是 agent-team 的界面层，版本线和主程序完全独立，主程序到 rc.1 了它还在 alpha.2。这种包不会跟着你升级，它自己走自己的。

## 顺手验了一遍

升级完之后我在网页版开了一个新会话，只发一句：调用 profile_ping，把返回的 JSON 原样贴出来。

十秒完成，返回的开头是 `"profile": "web", "dshVersion": "0.2.2-rc.1", "plugins": [` 后面跟着 212 个插件名。

这 212 行就是我这个 profile 现在的真实构成，从 `cordis-plugin-timer` 一直到我自己写的 `dsh-plugin-profile-ping`。想确认升级到底有没有生效，看这一个数字比看任何界面都准。

顺带一提，这个查询本身花了 56.3K token，缓存命中 86%，上下文只用了 3%。开销主要在插件清单太长，不在推理。

![](/images/dsh-022rc/fig5.png)

## 一个查版本的经验

如果你也想追 dsh 的版本，有个坑得避开：子包的 `latest` 标签是滞后的。

比如 `dsh-experimental-client-ui-agent-team`，它的 `latest` 停在 `0.1.5-alpha.2`，`alpha` 停在 `0.2.2-alpha.3`，真正的新版本在 `next` 里。

`npm view <包名> dist-tags` 三个标签一起看，别只看 `latest`。主包现在 `latest` 是对的，子包不是。

## 我的判断

这类版本值不值得跟？我的答案是跟，但不用急。

代码零变化意味着风险接近零，升不升都不会坏你的环境。真正的收益不在 dsh 本身，而在那些跟着转正的插件：`ui-agent-team` 这种跨版本线的包，只有在整条线一起转 rc 的时候才会被放出来。

反过来说，如果你手上有个跑得好好的 alpha.3，没有任何插件被禁，那这次不用动。等 0.2.2 正式版出来再一次性跟上，中间这几天不会有新功能给你。

你升级的时候会先看发布说明，还是直接拆包比一遍？
