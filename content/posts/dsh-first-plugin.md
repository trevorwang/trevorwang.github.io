---
title: "给 dsh 写第一个插件：三个文件、一条命令，外加我踩的两个坑"
date: 2026-09-29T07:24:54+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "9 月 28 日这一晚，DeepSeek Harness 的插件列表一口气多了三款：侧边栏左下角的加密货币行情卡片 crypto-ticker、把跑着会话的工作区顶到列表最上面的 busy-workspace、以及给消息自动选思考档位的 router-laya。"
cover:
    image: "/images/dsh-first-plugin/cover.png"
    alt: "给 dsh 写第一个插件：三个文件、一条命令，外加我踩的两个坑"
    relative: false
---

## 一晚上冒出三款插件

9 月 28 日这一晚，DeepSeek Harness 的插件列表一口气多了三款：侧边栏左下角的加密货币行情卡片 crypto-ticker、把跑着会话的工作区顶到列表最上面的 busy-workspace、以及给消息自动选思考档位的 router-laya。

同一天晚上 dsh v0.2.0-rc.1 发布，0.2 系列的第一个候选版。release notes 里专门有两条和插件有关的：「改善插件管理界面、内置插件界面布局及交互，改善安装引导」，「创造模式完善插件开发指引，并提供体验技能」。

装插件是一行命令的事。自己写一个，要迈过的坎就具体多了：插件到底什么形态、配置怎么被组合起来、装上去没生效该去哪儿看。

这篇把官方那套开发者文档按我自己跑通的顺序重走一遍。结论放前面：写一个能加载的插件只要三个文件，把它装进去只要一条命令。

## 先分清两个概念：组合包和 profile

官方文档里有一句概括得很准：

> A bundle is an npm package that carries a configuration layer… A profile is a directory describing one launchable composition… Nothing is both.

翻译过来：组合包是附带一份配置层的 npm 包，回答「这个包贡献什么」；profile 是 `$DSH_HOME/profiles/<name>` 下的一个目录，回答「这套配置由哪些组合包、按什么顺序组成」。

你写的插件是一个组合包。用户把它装进自己的 profile，profile 再把基础层、你的层、用户自己的层依次叠起来。dsh 自己的 base 和 web 应用也都是这么一层层拼出来的，所以「一切都由插件组成」不是句口号。

## 第一个插件：三个文件

官方教程给的目录结构是这样：

```text
hello-plugin/
├── package.json       # 声明 dsh.bundle
├── cordis.patch.yml   # 这个包贡献的配置层
└── index.js           # patch 行引用的插件入口
```

`package.json` 的关键只有一段：

```json
{
  "name": "dsh-hello-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "index.js",
  "files": ["index.js", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } }
}
```

`index.js` 是 Cordis 插件的入口。Cordis 加载模块时用一个上下文调用 `apply`，插件通过这个 `ctx` 注册自己贡献的一切：

```js
export const name = 'hello-plugin'

export function apply() {
  console.log('[hello-plugin] plugin loaded!')
}
```

`cordis.patch.yml` 就是你贡献的那一行。注意这里的 `name` 写的是包名而不是文件路径，这样 Node 的模块解析才能从 profile 的 `node_modules` 里找到你：

```yaml
- insert:
    - id: hello
      name: dsh-hello-plugin
```

## 装上去，先别急着启动

`dsh plugin --profile <name>` 会把参数转发给 profile 目录里的 pnpm，所以 pnpm 的子命令都能用：

```sh
dsh plugin --profile demo add ./hello-plugin
```

第一次跑会顺便初始化 profile，把 `@deepseek-ai/dsh-base` 作为第一个组合包；因为你的包声明了 `dsh.bundle`，dsh 会把它追加进 profile 的 bundles 列表。

我在这里踩了第一个坑：我 `cd` 进 `hello-plugin` 目录里，又写了 `add ./hello-plugin`。dsh 按当前目录解析，于是依赖记成了 `link:/private/tmp/hello-plugin/hello-plugin`，多一层路径，启动直接报：

```
Error: dsh: cannot resolve profile bundle "hello-plugin" from the dsh installation or /tmp/dsh-home-test/profiles/demo
```

换成在父目录执行 `dsh plugin --profile demo add ./hello-plugin` 就对了。教程里那句「在包含 hello-plugin 的目录中安装」，指的是父目录。

## 验证层真的进来了

装完不用启动就能看配置是怎么叠的：

```sh
dsh --profile demo --dump-config
```

我这边输出的层标记只有两个，第二个就是我加的：

```
# == @deepseek-ai/dsh-base
# == dsh-hello-plugin
```

然后真启动：

```sh
dsh --profile demo
```

屏幕上出现了 `[hello-plugin] plugin loaded!`。从建目录到看见这行，一共三条命令：装、看一眼层、启动。

顺手记一个细节：我启动时环境里带着 `all_proxy=socks5://...`，dsh 直接提示 `all_proxy names a SOCKS proxy, which is not supported`，自己改成直连了。这个提示挺大方，但如果你在靠代理装 GitHub 依赖，它救不了你，那个是 pnpm 的活。

## 层的顺序，和一条最容易吃亏的规则

生效配置在空根之上按四层组合，后面的层按行覆盖前面的：

1. profile 的 `dsh.profile.bundles` 列表里的各个组合包 patch，按列表顺序
2. profile 自己的 `cordis.patch.yml`
3. home 级的 `$DSH_HOME/cordis.patch.yml`，各 profile 共享
4. 命令行上的每个 `--patch <path>` overlay，按 argv 顺序

规则里有一句值得单独拎出来：patch 会替换目标行的整个 `config` 值，而不是深合并各个键。所以你的 patch 想覆盖前面某一层时，必须把那一行需要的每个键重述一遍，只写改动的那一个会把别的键清掉。

另一个方向的好消息是：用户可以在自己 profile 的 `cordis.patch.yml` 里覆盖你的行，不用改你的包。所以默认值尽量给用户大概率会保留的那种。

## 让插件接受配置

配置不是从命令行传的，而是导出 `Config` 类型加一份同名的 Schemastery schema，默认值写在 schema 里：

```ts
export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
  maxRetries: Schema.number().default(3),
})
```

加载时 Cordis 会用 schema 校验配置、补齐默认值。不合法就加载失败，并且报错明确。

这里的约定是 house style：凡是不同部署可能取不同值的参数，都必须定义成配置字段。检验标准就一句，能不能在 `cordis.yml` 里改掉它而不用改代码。而且改配置会触发插件热替换，旧实例的注册会被自动清理，因为注册都是 effect。

## 从打印一行到干实事

打印一行只是证明链路通了。真正有用的是三类东西：给模型加工具、给界面加卡片、给宿主加能力。

给模型加工具在官方教程里只有十几行，关键是 `inject` 声明你依赖工具注册表，然后注册一个 `defineTool`：

```ts
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: { name: { type: 'string', required: true } },
    async execute(args) { return `Hello, ${args.name}!` }
  }))
}
```

那三款新插件展示了更重的做法。crypto-ticker 是纯前端卡片，零依赖免密钥，数据源按 OKX、Binance、CoinGecko 顺次降级；busy-workspace 拆成 host 和 client 两半，host 只持有一份设置记录并开一条自己的 JSON 路由，client 挂载时读一次，读不到就用默认值，也就是说只带 client 的组合同样能用；router-laya 更狠，除了一份 4 万多字节的 index.js，还带了一个 Python 服务目录，里面是本地微调的 ModernBERT 分类器，专门判断该用哪档思考强度。

这三种形态说明一件事：插件能挂的钩子不只「给模型一个工具」，前端卡片、宿主路由、甚至一个旁路服务都在射程内。

## 我实测的两个失败行为

官方 Cordis 教程说，插件加载失败会明确报错，不会只跳过该配置项；但对「模块解析不到」（路径或包名拼错）它补了一句例外，说这种情况只通过 logger 报告，不会让进程崩溃。

我按这个说法试了两遍，结果和文档不太一样。

让 `apply` 抛异常，进程确实崩，而且崩得很清楚，第一行就点名了插件条目：

```
Error: dsh: plugin tree failed to load: failed to apply loader entry hello (dsh-hello-plugin): apply exploded
```

把 patch 里的包名改成 `dsh-hello-pluginX`，进程同样退出，exit code 1：

```
Error: dsh: plugin tree failed to load: failed to import loader entry hello (dsh-hello-pluginX): Cannot find package 'dsh-hello-pluginX' imported from /tmp/dsh-home-test/profiles/demo/
```

也就是说，至少在 dsh 的启动路径上，包名写错是致命错误而不是「记条日志继续跑」。这对写插件的人是好事，拼错名字会立刻炸给你看。文档那句例外说的是 Cordis 的通用行为，两边的口径差值得留意，遇到「加了配置项但没反应」时，先确认自己看到的是哪一类失败。

## 分发给别人

别人怎么装你的插件，官方给了三条路，成本差别不小。

发布到 npm 最省事，`dsh plugin add your-package` 装的就是编译好的产物。交付 tarball 次之，`pnpm pack` 之后让用户装那个 `.tgz`。

从 GitHub 直装最方便也最容易翻车：git 安装拉的是源码不是产物，没有任何环节会跑你的 build，所以 TypeScript 包到手时没有 `lib/`，加载会失败。作者要在 `package.json` 里提供自包含的 `prepare` 脚本，用户还要显式授权 pnpm 执行它（pnpm 10 之后默认拒绝跑 git 依赖的构建脚本），把包键写进 profile 的 `pnpm-workspace.yaml` 才算放行。官方文档把这句话说得很重：那等于允许这个包的代码在安装时于你的机器上执行。

国内网络还有一道：pnpm 用 `git ls-remote` 解析 `github:` 依赖，只走 HTTPS、不读系统代理，直连经常超时。crypto-ticker 的 README 给了一条实测可用的绕法，换成 codeload 的 tar 包地址，全程不经过 git：

```sh
dsh plugin --profile web add https://codeload.github.com/<owner>/<repo>/tar.gz/refs/heads/main
```

## 最后

如果只是因为插件列表一晚上长三款而心动，最小可行的路径就三步：建一个目录放那三个文件，用 `dsh plugin --profile <你的profile> add ./你的插件目录` 装上，再用 `--dump-config` 确认层出现了。

我第一次跑通大概花了二十分钟，其中一半时间花在一个路径上。

你打算给 dsh 挂什么？是给模型加个工具，还是像 crypto-ticker 那样往侧边栏塞一张卡片？评论区聊聊你的想法，也许下一篇就写它。

参考资料：
- dsh 插件开发教程（打包与安装）：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/user/develop/basic/publish.zh.md
- 从零写一个工具插件：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/user/develop/basic/tool.zh.md
- Cordis 教程第一章：https://github.com/deepseek-ai/deepseek-harness/blob/main/docs/cordis-tutorial/01-first-plugin.zh.md
- 插件注册表 awesome-dsh-plugin：https://github.com/awesome-dsh-plugin/awesome-dsh-plugin
- dsh v0.2.0-rc.1：https://github.com/deepseek-ai/deepseek-harness/releases/tag/dsh-v0.2.0-rc.1

验证边界：文中的三个插件文件、安装命令、`--dump-config` 层输出、启动日志和两个失败行为，都在本机 dsh 0.1.6-alpha.2 上真跑过（隔离在 `DSH_HOME=/tmp/dsh-home-test`，没动用户正在跑的实例）；官方文档引文逐字来自仓库。router-laya 的判定服务细节取自它的插件注册表条目，我没有部署它。
