---
layout: post
title: "把任意 AI Agent 打造成鸿蒙开发工具台"
date: 2026-10-13 09:12:18 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/harmony-ai/cover.png
---

![把任意 AI Agent 打造成鸿蒙开发工具台](/assets/images/harmony-ai/cover.png)

## 换一个 agent 解决不了的问题

通用 agent 写 ArkTS 会翻车，原因就三个洞。它不认识 ArkTS 的装饰器和构建命令；它手里没有一份能查的官方文档，只能凭印象编 API；就算代码写对了，它也不知道怎么把包构建出来、装进设备、送到应用市场。

洞可以一个个堵。华为和社区把堵洞的东西做成了可以外挂的工具，从零装到能用是八步。这篇按顺序走一遍，每一步给命令，也给该看到的输出。最后拿 DeepSeek Harness 把三样接进一个真 agent，看它到底认不认。

## 要装的是四件东西

DevEco Studio 或者它的命令行工具是地基，构建、设备、签名这些真活都在它身上。@deveco/deveco-cli 是工具集，命令名 devecocli，负责把能力装进别的 agent。

两个 MCP 分工不一样。远程的开发者知识 MCP 管查文档，本地的 deveco-mcp 管 .ets 和 C、C++ 的语法检查。最后一件是 agc-cli，社区写的上架命令行，把 AGC 后台的操作搬回终端。

另外先备一个华为开发者账号，装地基和生成签名都要用。

## 第一步：装地基

去开发者联盟的下载中心（developer.huawei.com/consumer/cn/download/），登录后选 macOS 版。Apple Silicon 的机器要的是 macOS ARM 那一档。

装完记一下路径，默认是 /Applications/DevEco-Studio.app。这一步不能跳，devecocli 后面所有的真活都要它。

不想装整个 IDE 的话，另一条路是同一页面上的命令行工具。体积小得多，但官方专门提了一句：

> Set `DEVECO_CLI_CLT_PATH` to the Command Line Tools root when DevEco Studio is not installed; CLT is not discovered automatically from PATH or default installation directories.

它不会自己从 PATH 或者默认目录里被发现，必须手写环境变量指过去：

```bash
# 装了 IDE
export DEVECO_CLI_STUDIO_PATH=/Applications/DevEco-Studio.app
# 只装命令行工具
export DEVECO_CLI_CLT_PATH=$HOME/command-line-tools
```

写进 ~/.zshrc 才会一直生效。Studio 首次启动会引导下载 HarmonyOS SDK，这一步要联网，也慢，别在没网的地方干等。

## 第二步：装 devecocli 工具集

两个 npm 包一起装：

```bash
npm install -g @deveco/deveco-cli @deveco/deveco-code
```

装出来两个命令。deveco 是 Agent 本体，基于 OpenCode 定制；devecocli 是工具集。两个包 255 个依赖、267 MB。

装完先验一下，别急着往下走：

```bash
devecocli --help
devecocli device list
```

第二条会列出连着的设备。如果它报 DevEco Studio 找不到，问题不在这一条命令上，回头查第一步那个环境变量。

## 第三步：把技能包装进你的 agent

这是「任意 agent」那一半的答案，一条命令：

```bash
devecocli init --skill --agent dsh,cursor,codex
```

不给 --agent 就是装到所有能识别的宿主。想装到指定目录用 --path，要覆盖已有配置加 -f。

装进去的东西长这样：

```bash
node_modules/@deveco/deveco-cli-common/SKILL.md
```

24,690 字节，开头的触发条件写得很直白：

> Use 'devecocli' for HarmonyOS apps (scaffold, build, run, debug, devices, emulators, logs, docs, UI inspection, 鸿蒙开发文档, skills). Use it if workspace has build-profile.json5 or oh-package.json5, or user mentions HarmonyOS, 鸿蒙, DevEco, ArkTS, ArkUI ...

agent 读到它之后，碰到鸿蒙工程或者提到鸿蒙，就会自己想到用 devecocli。同一个包还带着一份离线文档语料，docs.zip 有 40.5 MB，解开是 31,477 个文件，另加 15.2 MB 的检索索引。`devecocli docs search` 查的就是它，不联网也能用。

支持的宿主是官方给的名单：atomcode、codebuddy、claude-code、codex、cursor、dsh、deveco、opencode、pi、qoder、trae-cn，十一个。名单外的宿主也有条土路，技能包说到底就是一个 Markdown 文件加一份文档语料，把它放进那个宿主的 skill 目录就是同一回事。

想批量装或者查已经装了什么，走 skills 子命令：

```bash
devecocli skills list
devecocli skills add --all
```

## 第四步：装本地那个语法检查 MCP

```bash
devecocli init --mcp
```

它配的是 deveco-mcp，一个 stdio MCP，只暴露一个 check 工具，专门给 .ets、C、C++ 做语法检查。接进 agent 之后，它每改完一轮代码先自查一遍，不用等整包构建才报错。官方在说明书里还叮嘱了一句，一轮编辑跑一次就行，不要每次编辑都跑。

官方另有一句限定：所有支持的宿主都能配全局和项目级 MCP，只有 qoder 不支持项目级。

这个服务能手动起，调试的时候用得上：

```bash
devecocli serve mcp
```

它认几个环境变量，PROJECT_PATH、DEVECO_PATH，还有调 Node 内存上限的 NODE_MAX_OLD_SPACE_SIZE。

## 第五步：接官方知识 MCP

这一件最通用，任何支持 MCP 的客户端都是往配置里塞一段：

```json
{
  "mcpServers": {
    "harmonyos_developer_knowledge": {
      "url": "https://connect-api.cloud.huawei.com/api/developerknowledge/mcp",
      "type": "http"
    }
  }
}
```

DevEco Code 自己带 MCP 管理命令，一行就够：

```bash
deveco mcp add harmonyos_developer_knowledge \
  --url https://connect-api.cloud.huawei.com/api/developerknowledge/mcp
```

跑完它会自己报状态：

```
●  ✓ harmonyos_developer_knowledge connected
```

它只有两个工具。searchDocuments 按关键词搜，返回匹配的文本块和文档标识；getDocumentsById 拿标识换全文，一次最多十篇。成本要有数：搜一次三千字符左右，拉一篇全文三万五千字符，别让它一次拉十篇。官方说这份语料跟官网准实时同步，延迟不超过 30 分钟，这是这条路跟「让模型凭记忆答题」最要紧的差别。

## 以 DeepSeek Harness 为例：三件东西怎么落到一个真 agent 上

前面几步讲的是做什么，这一节拿 dsh 走一遍落到哪。它正好在官方那份宿主名单里，所以三件东西都能自动接。

技能包一条命令，落到 dsh 自己的技能目录：

```bash
devecocli init --skill --agent dsh
ls ~/.dsh/skills
```

目录结构跟它已有的技能一模一样，一个目录一份 SKILL.md，文件开头就两行 frontmatter，写 name 和 description。这台机器的 ~/.dsh/skills 里原本已经躺着七个 memorix- 开头的技能，deveco-cli 进来之后跟它们并列。

MCP 要写进 dsh 的配置文件 cordis.patch.yml。它的配置是一层层的 profile patch，MCP 走它自己的 mcp-client 插件，一个服务一条记录。知识 MCP 写进去是这样：

```yaml
- insert:
    - id: mcp-harmonyos-knowledge
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: harmonyos_developer_knowledge
        transport: streamable-http
        url: https://connect-api.cloud.huawei.com/api/developerknowledge/mcp
```

本地那个语法检查 MCP 只换 transport 和启动命令：

```yaml
    - id: mcp-deveco-syntax
      name: "@deepseek-ai/dsh-mcp-client"
      config:
        serverName: deveco
        transport: stdio
        command: devecocli
        args: [serve, mcp]
```

它只认两种 transport，stdio 是本地起的进程，streamable-http 是服务端。几个默认值值得知道：单次工具调用超时 60 秒，服务端下发的指令上限 32,768 字节，连接断了会自己重连，最多连试十次。

改完不用重启去猜，dsh 有 dump 命令，能把合成后的配置整个打出来：

```bash
dsh --profile <你的 profile> --patch ./hw-mcp.yml --dump-config
```

配置合成进 profile 树之后，工具名就真的出现在它的工具表里了，我拿一个隔离的 DSH_HOME 跑过一遍：

```
mcp__harmonyos_developer_knowledge__searchDocuments
mcp__harmonyos_developer_knowledge__getDocumentsById
```

命名规则是 mcp 加两个下划线加服务名加工具名，所以同一个 agent 里挂好几个 MCP 也不会撞名。到这一步，你在 dsh 里问一句鸿蒙的接口怎么写，它会先去查官方文档，而不是凭印象编。

![](/assets/images/harmony-ai/fig-dsh.png)

## 第六步：登录与签名

要真机跑，这一步绕不过去。

```bash
devecocli auth login
```

它会开浏览器走华为开发者账号的 OAuth。登录状态用 `devecocli auth status` 看，加入的团队账号用 `devecocli auth team list` 列。然后生成签名：

```bash
devecocli signature generate --product default
```

顺序官方给得很清楚，auth login、signature generate、build、run，一步都不能少。漏了哪步都有对应的报错：没登录会说 `Not logged in. Run devecocli auth login first`，令牌过期说 `Invalid AccessToken. Sign in and try again`，签名换了说 `error:install sign info inconsistent`，这时候 run 加 --uninstall，或者签名重新生成时加 --force。

## 第七步：装 agc-cli

上架走 AGC，以前只能在浏览器里一页页点。这个开源命令行把接口整理成一张可检索的注册表，实测 159 个接口、13 个家族，每条都带方法、路径、必填参数，还有一条指回华为官方文档的 sourceUrl。

Homebrew 那条路要先信任它的 tap：

```bash
brew tap createitv/tap
brew trust createitv/tap
brew install agc-cli
```

或者直接下官方 Release 的预编译二进制，有 darwin_arm64 的包，核完哈希再用：

```bash
shasum -a 256 agc-cli_0.2.0_darwin_arm64.tar.gz
./agc version
```

凭据有三种配法，走一种就行：

```bash
agc auth login --client-id <id> --client-key <key>
agc auth login --api-client-file client.json
agc auth login --service-account-file sa.json
```

`agc auth check` 看当前生效的是哪一套，`agc auth list` 列已存的所有档，`agc auth token` 给当前这档签一个临时令牌。

它默认只预演，真发之前你会先看到请求长什么样：

```bash
agc publishing app-info-query --invoke --query appId=123456789
{"data":{"method":"GET","url":"https://connect-api.cloud.huawei.com/api/publish/v2/app-info?appId=123456789&lang=zh-CN","dryRun":true}}
```

确认没问题，再显式加 --dry-run=false 才真发。每个接口的返回里还带一段 affordances，写着下一步能跑哪条命令。agent 不用背命令树，顺着返回往下走就行。

## 第八步：跑一遍，看它自己验货

装完这些，从工程到设备是一条直链：

```bash
devecocli build
devecocli emulator list
devecocli emulator start "Phone"
devecocli run
```

模拟器第一次用要先接受协议，交互式是 `devecocli emulator license`，要 TTY，脚本里用 `devecocli emulator license accept`。镜像下载失败别连着重试，官方说得很直白，这条交给用户手动跑。

跑完看工程里多出来的两个目录。`.verify` 放校验产物，`docs/screenshots` 放截图，后面这个是自动跑出来的：

```bash
docs/screenshots
  01-home.png                          首页
  03-quick-entry.png                   快捷记账
  05-stats-mpchart.png                 统计图表
  06-stats-retry.png                   统计失败后重试
  08-trip-edit.png                     行程编辑
  09-home-empty.png                    空态
  10-real-device-home.png              真机
  11-real-device-trip-detail-finish.png
  12-light-home.png                    浅色
  15-dark-home.png                     深色
  19-dark-to-system-fixed.png
```

编号从 01 排到 19。里面有两张真机截图，说明它把包装进设备真跑过；浅色、深色各一套，说明主题切换也在它的流程里；还有空态和统计失败后的重试态，说明它不满足于走顺路，会把边角的状态一个个摆出来。这些都不是手截的。

这一步的意义在闭环。以前 agent 写完代码交给你，好不好看得你自己去看；现在它能自己看到结果，截图落成文件，下一轮改什么就有依据。这也是我更愿意把 devecocli 归到「工具」而不是「客户端」的原因：它不是替你写代码的那只手，它是让 agent 能自己验货的那只眼。

![](/assets/images/harmony-ai/fig-workbench.png)

## 每步一条验证命令

装完别靠感觉，一条条过：

```bash
devecocli --help                  # 工具集在
devecocli device list             # 找得到设备
devecocli docs search ArkTS       # 离线文档在
devecocli skills list             # 技能包装上了
deveco mcp list                   # 知识 MCP 连上了
devecocli auth status             # 登录状态
agc auth check                    # AGC 凭据配好了
agc endpoints                     # 注册表读得到
```

哪条不对，问题就在它对应的那一步，不用到处翻。

## 一点判断

四件东西里我会先装哪件？先装地基，因为它卡着后面全部。但如果你只想花二十分钟试水，那就先接远程的知识 MCP，一个 URL、不用设备、不用账号，堵的又是最致命的那个洞。

顺序别倒过来。技能包和本地 MCP 都挂在 devecocli 上，devecocli 的真活又挂在 DevEco Studio 上，地基不牢，后面每一步都会撞同一句话。

你现在让自己常用的 agent 写鸿蒙代码吗？它最常在哪一步翻车？评论区聊聊。
