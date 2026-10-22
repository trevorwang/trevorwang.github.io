---
layout: post
title: "DeepSeek Harness 官方桌面端：没有下载页，355MB 的包一直在发"
date: 2026-09-25 10:05:13 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/dsh-desktop-official/cover.png
---

![DeepSeek Harness 官方桌面端：没有下载页，355MB 的包一直在发](/assets/images/dsh-desktop-official/cover.png)

## 官方站点没有下载页，但包一直在发

`@deepseek-ai/dsh-desktop` 在 npm 上是 404，官方仓库最近五个 release 的附件数全是 0，官网整页搜不到「下载」两个字。按这三条查下去，你会得出一个结论：官方没有桌面端。

但包一直在发。它不在下载页上，地址长这样：

```
https://download.deepseek.com/dsh-desk/feeds/mac-arm64/nightly-mac.yml
https://download.deepseek.com/dsh-desk/feeds/win-x64/nightly.yml
```

它是 electron-updater 的更新源清单，也就是**已安装的桌面端自己去拉更新的那个文件**。官方从来没打算让人手动点它，所以没有页面、没有入口、目录列表也是 404，只能拿着准确地址访问。

## 那个 YAML 里写了什么

mac-arm64 那份，全文只有 518 字节，核心就这几行：

```yaml
version: 0.1.7-rc.2
files:
  - url: >-
      https://download.deepseek.com/dsh-desk/bin/mac-arm64/deepseek-harness-0.1.7-rc.2-mac-arm64.zip
    sha512: >-
      aOfxtRvFqTRRp3zqu6nXNgILgVdHDPyfslhUVNnU1wMYw1dY1vTelptJeAmUDt7G822SCqHzxtPwMmvCGekveA==
    size: 372794444
releaseDate: '2026-09-24T14:10:00.562Z'
```

左边是清单原文，右边是我本机验出来的：

![官方更新源清单，以及本地校验结果](/assets/images/dsh-desktop-official/02-verification.png)

Windows 那份指向 `deepseek-harness-0.1.7-rc.2-win-x64.exe`，288245480 字节。

有个细节挺说明问题：这份 mac 清单的 `releaseDate` 是 2026-09-24T14:10:00.562Z，而 GitHub 上 `dsh-v0.1.7-rc.2` 的发布时间是 `2026-09-24T14:10:21Z`，差 21 秒。桌面端不是"隔几天补个包"，它跟命令行版本是同一次发布流程产出的。

我 HEAD 了两个包，都是 HTTP 200，真实体积也对得上：

- mac-arm64 zip：355MB，最后修改 09-24 14:20
- win-x64 exe：274MB，最后修改 09-24 14:12

后来我把 mac 那份**真的下下来了**，372794444 字节，和清单里的 `size` 一个字节不差；算出来的 sha512 也和清单里那串一模一样。解压出来是 `DeepSeek Harness.app`，**9535 个文件、解压后 1.02GB**。355MB 压成 1GB，这就是"自带整套运行时"的代价。

## 只发两个平台，还只有 nightly 一条通道

我把能猜的路径都试了一遍：

```
feeds/mac-arm64/nightly-mac.yml   → 200
feeds/win-x64/nightly.yml         → 200
feeds/mac-x64/nightly-mac.yml     → 404
feeds/win-arm64/nightly.yml       → 404
feeds/linux-x64/nightly-linux.yml → 404
feeds/mac-arm64/latest-mac.yml    → 404
```

macOS arm64 和 Windows x64，就这两个。仓库文档里写的打包目标其实是三个（`mac-arm64`、`mac-x64`、`win-x64`），但 Intel Mac 那份没发布出来。

`latest-mac.yml` 也没有，这个不是遗漏。仓库文档里写着：每个版本发布 `nightly.yml` 或 `nightly-mac.yml`，稳定版本还会额外发布指向相同产物的 `latest.yml` / `latest-mac.yml`。只有 nightly，说明官方桌面端现在还没有稳定版。

再看版本历史就更清楚了。同一个命名规则往前翻：

```
0.1.7-rc.2     → 200
0.1.7-rc.1     → 200
0.1.7-alpha.2  → 404
0.1.7-alpha.1  → 404
0.1.6-alpha.2  → 404
0.1.5-rc.3     → 404
```

**官方桌面端是从 0.1.7-rc.1 才开始发的**，也就是 9 月 23 号。往前全是 404。之前那些文章里的"桌面版"，都是社区自己套的壳。

## 355MB 里装了什么

一个 Electron 壳子做到 355MB，肯定不是壳的事。

官方桌面端把整套运行时都带上了：Python、Node.js、pnpm 随包分发，文档里那句要求是"应用必须能够在没有系统 Node.js 或 pnpm 的机器上运行"，做法是把 dsh 跑在 Electron 的 `ELECTRON_RUN_AS_NODE` 模式下，包操作全用手里的内置 pnpm。

那套 Python 也不是摆设，装了 numpy、pandas、python-docx、python-pptx、openpyxl、Pillow、lxml、XlsxWriter，随包注册 `office-docx`、`office-pptx`、`office-xlsx` 三个技能，Office 任务默认走随包的 LibreOffice。首次调用时离线装到 `$DSH_HOME/dsh-runtimes/dsh-primary-runtime`，之后不碰你系统的 Python，也不改 PATH。

我前两天把源码拉下来构建过一次，跑起来第一件事是占住 19387 端口，和 Web 版的 3080 分开，两个能同时开着。

## 一个发布号，锁死一套版本

这是我认为整个桌面端最值得学的决定，写在官方的设计记录里。

Electron 壳和它内置的 dsh 是两个独立的包。GUI 协议又绑定 Web 客户端和后端版本，所以只要允许两边各自选版本，就会出现"未经验证的壳、客户端、后端与插件组合"，而且判断不了更新到底可不可用。

官方的做法是：**一个 Desktop 发布号同时标识 Electron 产物和它精确的 `@deepseek-ai/dsh`、`@deepseek-ai/dsh-desktop-host` 依赖。发布过程不能在构建或运行时挑另一个核心版本。**

后果就是那句有点绕但很硬的话：**即使壳代码完全没变，更新 dsh 也必须产生一次新的 Electron 发布。**

这解释了我前面看到的两个现象：`releaseDate` 和 GitHub 发版只差 21 秒；rc.1 和 rc.2 各有一个 355MB 的包。哪怕这两个版本之间 Electron 一行没改，桌面端也得重新打一遍。

## 桌面 profile 是保留的，CLI 碰不了

桌面端和 `npx` 装的 dsh 共用同一个 `.dsh` 目录，但有两件事必须分开：

- 共享的：会话、设置、凭据、工作区、存储
- 不共享的：可执行包、lockfile、`node_modules`、插件激活状态、包管理器配置

所以桌面端有自己独占的 profile，位置是 `~/.dsh/profiles/desktop`。而且 CLI 保留 `desktop` 这个名字的所有大小写变体。你没法用命令行去启动它、dump 它的配置、或者管它的插件，请求会被直接拒掉。桌面端归 Electron 管。

别的细节也能看出这条线画得多认真：Electron 在恢复 profile 或启动 Host 之前先拿一个进程生命周期单实例锁，后续启动只会聚焦或重建窗口，不碰 profile 状态；渲染进程是 `nodeIntegration: false`、`contextIsolation: true`、`sandbox: true`，preload 只暴露启动就绪、原生目录选择、主题同步这几个能力，不暴露原始 `ipcRenderer`、文件系统、shell 和 pnpm 参数。

那个私有的 Desktop Host 包不发到 npm，这也是为什么我在 npm 上搜 `@deepseek-ai/dsh-desktop` 会 404。它本来就不该被单独装。

## 更新和卸载：两道校验，一张不动的 ~/.dsh

更新走的是 electron-updater 那一套，但官方加了两道保险：清单里带 `sha512`，Windows 那侧还有签名校验，`app-update.yml` 里存着预期的发布者身份，下载来的清单不能自己改这个身份。

安装过程也照顾了"装到一半断电"这种事。Windows 安装器会先把新版本解压到安装目录旁边，再退出旧应用，然后用同卷目录改名完成替换。替换成功前旧目录一直留着，解压失败则旧版原样不动，替换失败还会尝试把旧目录恢复回来。真解压失败时，7-Zip 的完整错误会写进 `%LOCALAPPDATA%` 下的更新缓存目录，弹窗里只显示第一行，旁边给一个"复制错误信息"按钮。

macOS 那边要求更硬：App 和 DMG 各走一遍公证，两路并行，都成功了产物才会移进最终目录。

卸载这一侧有个细节我挺喜欢：Windows 卸载程序会删掉 Electron 的用户数据、`%APPDATA%` 下的产品目录和更新下载缓存，但不碰 `~/.dsh`，你的会话、设置、凭据、插件全部留着。装回去就是原来的样子。

## 想试的话

macOS（Apple Silicon）：

```
https://download.deepseek.com/dsh-desk/bin/mac-arm64/deepseek-harness-0.1.7-rc.2-mac-arm64.zip
```

Windows x64：

```
https://download.deepseek.com/dsh-desk/bin/win-x64/deepseek-harness-0.1.7-rc.2-win-x64.exe
```

建议下完先对一下 sha512，清单里就给了：

```sh
openssl dgst -sha512 -binary deepseek-harness-0.1.7-rc.2-mac-arm64.zip \
  | openssl base64 -A
```

mac 那份应该输出 `aOfxtRvFqTRRp3zqu6nXNgILgVdHDPyfslhUVNnU1wMYw1dY1vTelptJeAmUDt7G822SCqHzxtPwMmvCGekveA==`。

三件事得说清楚：

第一，这是 **nightly 通道**，官方自己的定位就是开发者预览，不是给日常用的稳定版。

第二，**我只验到"这个包是真的"这一步**。清单可读、两个包 HTTP 200、体积和 sha512 都对得上、解压出来的东西和文档描述一致，这些都是我本机跑出来的。但**我没有安装它、没有启动它**，所以"装上去能不能跑"我不替你保证；macOS 的签名与公证我也没有重新验一遍（官方文档里写的是 App 和 DMG 各走一遍公证、两路都成功产物才会发布）。装不装、装哪个版本，你自己判断。

第三，它不是唯一的选择。社区那几个壳子也还在维护，各自跟的 dsh 版本不一样：dshdesktop 建在 `0.1.5-rc.2` 上，官方已经到 `0.1.7-rc.2`，中间隔着语音输入和账号登录。要新东西就跟官方，要开箱即用、不想折腾签名和公证就去社区那边。

## 小结

官方从来没把桌面端当"要下载的软件"来做，它是**更新通道的一部分**，所以只存在于 YAML 清单里，只存在于 `bin/` 和 `feeds/` 这两个目录的约定里。

查一个东西"有没有"，值得先问自己一句：如果它压根不该被手动找到，它会放在哪。

链接：

- mac 更新源：https://download.deepseek.com/dsh-desk/feeds/mac-arm64/nightly-mac.yml
- win 更新源：https://download.deepseek.com/dsh-desk/feeds/win-x64/nightly.yml
- 仓库：https://github.com/deepseek-ai/deepseek-harness
- 官方站点：https://www.deepseek.com/harness/
