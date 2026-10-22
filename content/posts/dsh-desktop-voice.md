---
title: "DeepSeek Harness 官方桌面端：支持账号直接登录，语音输入跑在你自己 CPU 上"
date: 2026-09-24T17:33:14+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "dsh 是个 Web 应用：npx 起服务，占 3080 端口，浏览器开个标签页，在里面和 agent 干活。能用，但切个窗口、关掉浏览器，它就得重新找回来，始终像住在别人家里。"
cover:
    image: "/images/dsh-desktop-voice/cover.png"
    alt: "DeepSeek Harness 官方桌面端：支持账号直接登录，语音输入跑在你自己 CPU 上"
    relative: false
---

dsh 是个 Web 应用：`npx` 起服务，占 3080 端口，浏览器开个标签页，在里面和 agent 干活。能用，但切个窗口、关掉浏览器，它就得重新找回来，始终像住在别人家里。

0.1.7 系列塞了两件事，把这个别扭解决了一半：官方桌面端，和客户端的语音输入。语音是本地推理，你的声音不出这台机器。

## 桌面端把运行时一起打包了

看到"官方桌面端"，我第一反应是 Electron 包一层 WebView。翻完架构文档发现不止这个。

先把界面放这儿，第一次启动是这个 600×700 的欢迎窗口，有自己的原生窗口控件：

![DeepSeek Harness 桌面端欢迎窗口](/images/dsh-desktop-voice/01-welcome.png)

`apps/desktop` 是 Electron 壳，但它把整套运行时都带上了：Python、Node.js、pnpm 随包分发。文档里有句话，"应用必须能够在没有系统 Node.js 或 pnpm 的机器上运行"，做法是把 dsh 跑在 Electron 的 `ELECTRON_RUN_AS_NODE` 模式下，包操作全用手里的内置 pnpm。

这套 Python 装了 numpy、pandas、python-docx、python-pptx、openpyxl、Pillow、lxml、XlsxWriter，随包注册 `office-docx`、`office-pptx`、`office-xlsx` 三个技能，Office 任务默认走随包的 LibreOffice。`load_workspace_dependencies` 首次调用时离线装到 `$DSH_HOME/dsh-runtimes/dsh-primary-runtime`，之后不碰你系统的 Python，也不改 PATH。

端口用的是 19387，和 Web 的 3080 分开，两个能同时开着。更新上它把壳、匹配的 dsh 运行时和 pnpm 打成一个已签名单元。文档给的理由很直白：独立更新会让运行时版本选择脱离 Desktop 发布，那就没意义了。

Electron 在访问任何 profile 之前先拿单实例锁，并独占 `$DSH_HOME/profiles/desktop`。CLI 和桌面端共享 `$DSH_HOME` 下的产品数据，但可执行包、插件、lockfile 和 `node_modules` 各管各的，所以插件得分别装。

## 账号可以直接登录，不用手动填 API Key

前面那张欢迎窗口上有两条路：Sign in，或者自己填 API Key。我第一反应是前者大概还没通，因为它文档的"已知限制"里就写着「账号登录尚未接入；登录按钮禁用」。

实际是通的。这是登录之后的设置页，账号、手机号、余额都在里面：

![桌面端用账号登录后的设置页](/images/dsh-desktop-voice/05-account-login.png)

登录走系统浏览器：Host 负责 PKCE 和临时本机回调，凭证在进入工作区之前就存好了，之后浏览器再跳到平台的完成页，授权链接还带着当前桌面端的主题（`theme=light` 或 `theme=dark`）。设置里的账号页可以退出。

所以文档那句"已知限制"是过时的。同一份 README 的另一个章节其实已经把整套登录流程写全了，只有限制清单没跟着更新。这个项目还在开发者预览、迭代很快，文档有时跟不上代码，读它的文档得留这个心。

## 语音输入的模型在你机器上

0.1.7-rc.1 的发版说明里有一行：

> 实验性语音转写首次使用时准备本地模型；支持自动或手动选择 Hugging Face 与 HF-Mirror 下载源，并在自动模式下尝试其他可用源。

实现上分了三个角色：服务定义负责路由具名 Provider，SenseVoice Provider 管本地推理，Remote 消费者服务浏览器。干活的是 SenseVoiceSmall ONNX 加 Silero VAD，跑在 Host 的 CPU 上，通过 sherpa-onnx 的 Node 原生包调用。那个包已经带了 ONNX Runtime，用户不用装 Python，不用编译器，也不用做模型转换。

精度默认 `int8`，能切 `fp32` 用参考权重。int8 约 239 MB，fp32 约 938 MB，另加运行时和 VAD 的开销。

用法上它不占新位置。麦克风按钮在模型选择器和发送按钮之间，点一下开始录，工具栏展开，停止后转写并插入草稿。波形画的是实测麦克风音量。取消、按 Escape 或者把页面藏起来，录音就丢掉。语言提示给了六个：自动、中文、英语、粤语、日语、韩语。

这是我打开插件之后截的输入框，麦克风就在模型选择器和发送按钮中间：

![打开语音插件后的输入框，麦克风在模型选择器和发送按钮之间](/images/dsh-desktop-voice/04-composer-mic.png)

## 本地推理的几条硬约束

本地推理很容易变成宣传词，这库在几处是硬约束。

最硬的一条是没有 Provider 回退。设计决策文档写得很清楚，本地失败不能授权把语音上传到云端服务，音频不会因为本地出错就悄悄换个 Provider 发出去。Provider 的选择是显式的，`SpeechProviderInfo` 里带着 `host-local` 或 `cloud` 的处理位置，以后加云端 Provider 也得走同一个注册表、带自己的凭据配置。

模型文件要过校验：按固定修订下载，校验大小和 SHA-256 才加载。Host 会并发发 HEAD 探 `huggingface.co` 和 HF-Mirror，谁先返回 2xx 就从谁那下；显式指定单一来源时不探测也不回退。完全离线的部署根本不公布下载源。

转写也不碰会话。录音和转写在用户提交之前都是临时数据，识别不启动智能体轮次，也不创建音频 Session 事件，用户自己那次提交才拥有最终进入模型的文字。切换会话或释放插件会让迟到的结果失效。

## 怎么开起来，和一个会让你白折腾的坑

先说坑，它真的会浪费你半小时。

npm 上这个包的 `latest` 标签还停在旧版本。我实测的：

```
$ npx -y @deepseek-ai/dsh@latest --version
0.1.5-rc.3
$ npx -y @deepseek-ai/dsh@next --version
0.1.7-rc.1
```

`0.1.5-rc.3` 是 9 月 22 号发的，语音输入在 0.1.7 系列里。所以照着仓库 README 敲 `npx @deepseek-ai/dsh web`，装到的是旧版，翻遍插件页也找不到语音。

要试就显式装 next：

```sh
npx -y @deepseek-ai/dsh@next web
```

语音 bundle 是 dsh 0.1.7-rc.1 的直接依赖，随包带，但默认禁用，要去插件管理页显式开启。插件页里它是这个样子的，归在 Official 分组，开关默认是关的：

![插件页里的 Voice input，默认关闭](/images/dsh-desktop-voice/02-plugins-voice-off.png)

拨开之后不会立刻下载 239 MB，而是先弹这个对话框，告诉你首次使用要下载并准备本地模型，并让你先去插件详情里看磁盘和内存估算：

![打开 Voice input 后弹出的设置提示](/images/dsh-desktop-voice/03-voice-setup-dialog.png)

这个先后顺序是刻意的。文档里说，在插件启用时就加载模型会让一个 UI 开关在你还没听写之前就吃掉启动时间和内存。

桌面端不在 npm 上。`apps/desktop` 的 package.json 标了 `private`，registry 也查不到，只有从源码起：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run dev:desktop
pnpm run start:desktop
```

我按这几步真跑了一遍：`pnpm install` 用了 27 秒，全量构建 94 秒能过，但中途撞了个本机工具链的坑。我机器上 Xcode 26.2 的 clang 和 ld 配的是 Command Line Tools 27.0 的 SDK，新 SDK 的 `.tbd` 里声明了 `arm64e.x1-macos`，老 ld 不认，于是任何原生编译都链接失败：

```
ld: tapi error: malformed file
/SDKs/MacOSX27.0.sdk/usr/lib/libSystem.B.tbd:4:20: error: unknown architecture
```

命令行加一句 `export SDKROOT=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX26.2.sdk` 就绕过去了，根治得让 Xcode 和 CLT 版本对齐。这个坑不只影响这个仓库。

顺带一提，CLI 里没有 `desktop` 子命令。`dsh --help` 给出的子命令是 `web`、`tui`、`headless`、`rescue` 这些，桌面端那个 profile 由 Electron 独占，CLI 启动不了也改不了。

## 该说清楚的限制

语音只跑 CPU。原生包覆盖 macOS arm64/x64、Linux glibc arm64/x64 和 Windows x64，Windows ARM64 和 Linux musl 还不是已验证的部署目标。

量化不等于等价。官方自己写的：公开英文和拼接的中英双语样本上，int8 与 fp32 存在用词差异，所以不把量化视为准确率等效。想省磁盘就 int8，要参考权重就 fp32。

验证记录里是 4.2 秒公开中文样本三次识别结果一致，紧接着一句"该短样本不代表完整准确率评测"。这个自我限定写得挺老实，也意味着别拿它当准确率结论。

另外，取消一次推理会丢掉预热进程，下次请求得重新加载。

桌面端的"三大平台"要打个折扣。中文媒体说三大平台支持，仓库里明确的打包目标只有这三个：

```
pnpm run package:desktop:mac:arm64
pnpm run package:desktop:mac:x64
pnpm run package:desktop:win:x64
```

文档原话是「Linux 不是受支持的 Desktop 发布目标」。macOS 两个架构加 Windows x64，不是三个操作系统。

整个项目还在开发者预览，README 第一段就警告"未来将出现破坏兼容性的变更"。

## 小结

这库把语音输入做成了一个可插拔的插件包，推理放在本机，还专门设计成本地失败时不会偷偷改用云端。在大家都愿意把录音传上服务器的年代，这个选择挺少见。

桌面端那边，Electron 外壳本身没什么可说的，它连 Python 和 pnpm 都随包带了，等于把环境问题整个收进安装包。

我这边的验证边界：版本号、CLI 子命令、打包命令、依赖关系、构建流程都是本机实跑或查注册表确认的。上面几张界面截图，是把源码拉下来以开发模式真跑起来截的，跑的是 0.1.7-rc.1-46a7f68、Electron 44.0.0，服务落在 19387；账号登录那张也是同一个开发版里登录成功后的实际状态。唯一没跑通的是真实录音转写那一段，它要先把 239 MB 的模型下下来，我留给你自己试。

---

仓库：https://github.com/deepseek-ai/deepseek-harness

文档：https://deepseek-harness.github.io/deepseek-harness/

语音输入子系统文档：`docs/subsystems/voice-input.zh.md`

桌面端架构文档：`apps/desktop/README.zh.md`

实测环境：macOS / Node 25 / dsh 0.1.7-rc.1 与 0.1.5-rc.3（版本号取自 npm 注册表与 CLI 输出）。
