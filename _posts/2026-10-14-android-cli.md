---
layout: post
title: "Agent 的官方命令行：Android CLI 1.0"
date: 2026-10-14 14:36:53 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/android-cli/cover.png
---

![Agent 的官方命令行：Android CLI 1.0](/assets/images/android-cli/cover.png)

## 先说一个断掉的地方

你让 agent 写了一段 Kotlin，它说改好了。然后呢？

你得自己开 Android Studio，等 Gradle 转圈，等模拟器起来，再用鼠标戳一下界面，看那句文案是不是真的变了。命令行里 agent 跑得飞快，到了看屏幕这一步就断了，剩下的活儿归你的眼睛。

Google 这次补的就是这一段。Android CLI 进了 1.0 稳定版，官方博客的原话是这样：

> 无论您是在 Android Studio、Antigravity 2.0、Antigravity CLI 中使用 Gemini，还是使用 Anthropic 的 Claude Code 或 OpenAI 的 Codex 等第三方代理，我们的使命始终如一：确保随时随地都能进行高质量的 Android 开发。

意思很直白：不管你用哪个 agent，官方都想让你别离开命令行。我在本机跑了一遍，从建工程一直跑到看屏幕。

## 它交付了什么

下面这份清单，我按「它能替你的 agent 少做什么」来说。

### 一个 android 命令

装上之后你得到 `android` 这一个命令，19 个顶层子命令，建工程、起模拟器、装应用、看屏幕都在里面：

```
auth completion create describe device docs emulator info init
install layout run screen sdk skills studio update help
```

以前 agent 要记 adb、emulator、gradle 三套参数，现在一个入口对着一个 SDK。它只往用户目录装一个 3.6 MB 的单文件程序，装完 `android info` 就能报出 SDK 路径和版本。

### 官方技能库

`android init` 这步有点意思。它没有任何参数，跑完却把一份 24,922 字节的 `android-cli` 技能写进了你机器上每一个 agent 的技能目录。我这边命中了 10 个：

```
~/.claude/skills          ~/.gemini/skills           ~/.agents/skills
~/.commandcode/skills     ~/.hermes/skills           ~/.junie/skills
~/.openclaw/skills        ~/.config/opencode/skills  ~/.pi/agent/skills
~/.gemini/antigravity/skills
```

Hermes 在里面，Claude Code、Codex、Gemini、OpenClaw、OpenCode 这些也都在。跑完建议看一眼它都往哪些目录写了东西，毕竟是往你的 home 里落文件。

技能库里现在有 24 个技能，`android skills list` 能列全。界面类的有 adaptive（自适应界面）、camerax（相机迁移）、edge-to-edge、wear-compose-m3（Wear OS）。

工程治理类的也有几个：play-policy-insights（政策解读）、restore-credentials、testing-setup、r8-analyzer。装上就一句 `android skills add <技能名>`，还能指定装给哪个 agent。

### 离线的官方文档库

`android docs search "edge to edge"` 第一次跑会下载索引，实测 5,319 条。返回的不是网址，是 `kb://android/...` 这种协议地址。

连技能里的示例代码都在里面，比如 `kb://android/agents/skills/wear/wear-compose-m3/references/material3/EdgeButtonSample.kt`。agent 不用再去搜博客，官方文档本地可查。

### 看屏幕这组命令

这组是整篇里最实用的。`screen capture --annotate` 会把界面上的元素标成 `#1`、`#2`，`screen resolve` 再把标号换成坐标：

```bash
android --no-metrics screen capture --annotate --output shots/12-annotated.png
android --no-metrics screen resolve --screenshot shots/12-annotated.png --string "tap #1 and #2"
# 输出：tap 72 31 and 835 30
```

agent 不必再算像素，指着 `#3` 说点这个就行。同一组的 `layout` 直接返回界面 JSON 树，每个控件的 `bounds` 和中心点都在。

### 借用 Android Studio 的语义分析

新增的 `android studio` 子命令有这些：`find-declaration` 找定义、`find-usages` 找引用、`analyze-file` 跑检查、`render-compose-preview` 渲染 Compose 预览、`version-lookup` 查依赖版本、`open-file` 在 IDE 里开文件。

等于 agent 能借到 IDE 攒了十几年的语义分析，不用靠正则猜。这条有门槛，要求 Android Studio Quail 1 或更高。

### 预约一台真机

`android device remote` 可以预约远程真机来跑，官方博客贴的例子是连着一台 Pixel 10 Pro。这条路要先 `android auth` 登录 Google 账号。

## 我在 macOS 上从零跑了一遍

能不能跑通才算数。下面这套是我本机实跑的命令，一条没省：

```bash
# 装：先把脚本下下来读一遍，别直接 pipe 进 shell
curl -fsSL -o /tmp/android-cli-install.sh \
  https://dl.google.com/android/cli/latest/darwin_arm64/install.sh
bash /tmp/android-cli-install.sh            # → ~/.local/bin/android，3,650,880 字节

android --no-metrics -V                     # 1.0.16500706，首次要过一次 Android SDK 条款
android init                                # 把技能写进各 agent 的技能目录
android --no-metrics skills list            # 官方技能目录，实测 24 个
android --no-metrics docs search "edge to edge"

# 端到端：建工程 → 起模拟器 → 出包 → 装上去 → 看屏幕
android --no-metrics create --name CliDemo \
  --application-id com.example.clidemo --min-sdk 36 -o ~/android-cli-repro/CliDemo
android --no-metrics emulator list          # 我这儿已有的 AVD：Medium_Phone_API_36.1
android --no-metrics emulator start Medium_Phone_API_36.1   # 等到就绪才返回，约 17 秒
cd ~/android-cli-repro/CliDemo
./gradlew :app:assembleDebug                # 首次：BUILD SUCCESSFUL in 1m 24s
android --no-metrics run --apks app/build/outputs/apk/debug/app-debug.apk
android --no-metrics screen capture --output shots/10-before.png
android --no-metrics layout
```

先说建工程。`create` 出来的工程是 Compose + AGP 9 + Kotlin 2.3.20，我都没去 SDK Manager 里点，它自己发现缺 `platforms/android-36` 就装上了。

`emulator start` 这个设计挺舒服：一直等到设备完全启动才返回，中间只打印剩余秒数，不用自己轮询 adb。我这边 17 秒就绪。

真正让我意外的是第二次。我改了一行文案，把数据源里那句 `listOf("Android")` 改成 `listOf("Android CLI")`，重跑一轮：

```bash
./gradlew :app:assembleDebug                # 增量：BUILD SUCCESSFUL in 8s
android --no-metrics run --apks app/build/outputs/apk/debug/app-debug.apk
# Delta Install: Sent 167.94 KB patch instead of 28.67 MB (1 APK, 99.43% savings)
```

第一次装传了 28.64 MB，第二次只传了 167.94 KB 的补丁，省了 99.43%。增量安装默认就是开的（`--use-delta-install` 默认 true），改文案、看效果这种来回折腾的活儿，省下的全是等待时间。

改完再用 `layout` 读界面树，文案已经换新的了：

```
应用窗口: CliDemo  焦点=True  边界=[0,0][1080,2400]
  - android.widget.TextView | text='Hello Android CLI!' | center=[220,136]
```

![](/assets/images/android-cli/fig-loop.png)

![](/assets/images/android-cli/fig-agent.png)

## 上手门槛和几个坑

最容易绊人的是 `run`。它的说明写「构建、部署、启动」，我这里不带 `--apks` 直接失败：

```
Error: No apks specified. Use --apks <path1,path2,...>
Component check failed
```

APK 已经构建好、`describe` 也刚定位到它，`run` 还是不肯自己动手。实际分工是出包归工程自己的 `./gradlew`，`run` 负责装和起。

还有一个更隐蔽的。包部署上去了，界面却没变，旧进程还活着，`run` 不会替你强杀。补一条 `adb shell am force-stop com.example.clidemo` 再跑就对了。

版本门槛也在。`android studio` 那组命令要 Android Studio Quail 1 或更高，版本不够报的是 `No running Studio instances found`，还附一句让你去官网下新版。

另外两点：`android init` 是往你的 home 目录里写文件，不是只读操作，机器上 agent 多的话建议先看清名单；子命令是单数 `device`，写成 `android device list` 会报 `'list' is not a valid command`。

还有 `android completion` 会往 shell 配置里写自动补全，不想动配置就别跑它。

遥测也得提一句。这个 CLI 默认会收数据，官方文档写了收什么：命中的子命令、用到的参数名，以及一部分固定取值，比如模拟器模板名和 agent 名（`GEMINI`、`CLAUDE`、`CODEX`）。不想要的加 `--no-metrics`，我上面所有命令都带着。

## 最后一句

这版没有另搞一套给 agent 用的流程，改的是原本那套东西：文档、技能、模拟器、真机、IDE 的语义分析，都换成了机器能读的接口。

细节上也照顾到了，像素坐标换成 `#3` 这种标号，UI 树直接给 JSON。

你手上那套工具链，敢不敢把「改代码到看屏幕」这一段整个交给 agent？评论区聊聊。
