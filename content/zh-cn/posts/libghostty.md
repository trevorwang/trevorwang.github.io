---
title: "JetBrains 终端换芯，AI agent 太能吃了"
date: 2026-10-28T13:35:29+08:00
draft: false
categories: ["公众号"]
tags: ["IntelliJ IDEA", "AI Agent", "终端"]
summary: "IDE 的终端换了引擎：用了 13 年的 JediTerm 让位给 Ghostty 的核心库，理由写在 AI agent 那行。我把新版安装包拆开，验了一遍。"
cover:
    image: "/images/libghostty/cover.png"
    alt: "JetBrains 终端换芯，AI agent 太能吃了"
    relative: false
---

在 IDE 的终端里用 Claude Code 这类 CLI agent，大概都碰到过同一件事。敲了一句话，想按 Shift+Enter 换行再补两句，结果半句话直接发出去了，agent 已经开始按残缺的指令干活。

JetBrains 在 10 月 27 日的博客里把这件事写进了理由里。从 2026.3 开始，他们家 IDE 的内置终端不再用 JediTerm，改跑 libghostty，也就是 Ghostty 那个终端模拟器的核心库。

> Shift+Enter starts a new line in your prompt instead of sending a half-written message.

一句「instead of sending a half-written message」，把过去十几年的默认行为说得很直白。

## 一个终端库，凭什么换掉用了 13 年的那个

JediTerm 是 JetBrains 自己的纯 Java 终端模拟器，2013 年 1 月开的仓库，一路维护到现在。它处理 SSH 和 PTY，够用，也很稳。

问题是它不支持后来那批现代终端协议。

官方给的判断是，TUI 应用越来越多地依赖 JediTerm 从来没支持过的协议，而这个缺口以前「感觉只是个小不便」，因为体验确实跟别的终端差点，但没差到值得报 bug 的程度。

变的是终端里跑的东西。Claude Code、Codex、Neovim、tmux、各种系统监控面板，这些工具压在终端上的强度，远不是一个 shell 提示符能比的。

libghostty 接过了解析和终端状态这两件事。IDE 那边继续管渲染，以及上层功能和集成。这个分工官方写得很清楚，不是整个终端重写。

顺带说一句 libghostty 自己的定位。它用 Zig 写，对外是 C ABI，Ghostty 的 macOS 版（Swift + AppKit）和 Linux 版（Zig + GTK4）都是它的消费者。Mitchell Hashimoto 在项目介绍里说得很坦白，这是他业余时间做的项目，不是任何人的全职工作。

## Shift+Enter 终于能换行，以前为什么不灵

终端里键盘事件的老协议只区分得出「按了哪个键」，区分不出 Shift、Ctrl、Alt 的组合，也不告诉你按键是按下还是松开。

Kitty keyboard protocol 把这些补上了。终端应用现在能分辨 Enter、Shift+Enter、Ctrl+Enter，还能要求接收按键释放和连发事件。

Vim 用的是更老的 xterm modifyOtherKeys，这次也一起支持了。

对天天在终端里跟 agent 打交道的人来说，这带来两个变化。换行就是换行，不会半句话飞出去；Ctrl、Shift、Alt 跟别的键组合出来的快捷键，能原样送到应用手里。

官方还提了一句细节。以前那个让 Shift+Enter 发 `Esc` + `CR` 的旧设置不需要了，设了也会被忽略。

## 切到 Dark 模式，终端里的工具为什么也跟着变

这件事的体验落差特别明显。

IDE 切到深色模式，终端里的程序是不知道的，它只能猜。猜错的结果就是深色背景配深色文字，或者在一个全黑的 IDE 里塞一块亮色终端。要修只能一个工具一个工具手工配。

现在终端会回答 dark/light 查询，也就是 `CSI ? 996 n`，答案来自当前 IDE 主题。切主题的时候，通过 mode 2031 发一条通知出去。

Claude Code、OpenCode、Pi、Neovim、tmux 都订阅了这类通知，所以大部分工具能跟着 IDE 一起切换，IDE 这边不用调任何设置。

调色板查询也扩了。JediTerm 只处理 OSC 10/11，新的终端还能回答 OSC 4 和 Kitty 的 OSC 21。程序可以用 OSC 4 设置 16 到 255 号扩展色，0 到 15 号仍归 IDE 主题管。

## 那些乱七八糟的控制字符，以前为什么显示成方块

官方说新的引擎处理了近 50 个 JediTerm 忽略或误读的控制序列。挑三个能对上日常体验的：

REP（`CSI Ps b`）是 ncurses 用来压缩连续重复字符的，7 个以上相同字符走这条。以前有些行渲染出来缺字，就是它没被处理。

冒号分隔的 SGR 颜色（`38:2::r:g:b`），旧终端直接把冒号打印出来，还把数字合并成了错误的属性。你看到过终端里冒出一串 `38:2::255:0:0` 这类东西，就是这里的问题。

未知的 APC、SOS、PM 字符串，以前会被当成普通文本打出来，现在静默丢弃。

还有一批边角能力，左右边距、保护区域、tab stop 控制、insert mode、XTSAVE/XTRESTORE、Kitty 的滚动清除、X10 鼠标上报。光标闪烁模式以前是不管设不设都一直闪，现在会听。

![](/images/libghostty/fig4-changes.png)

## 我把这个库单独拽出来，跑了一下

上面那些是官方口径。下面这段是我在自己机器上验的。

先做新旧对照。本机装的是 IntelliJ IDEA 2026.2.3，`Contents/lib` 下能找到 `intellij.libraries.jediterm.core.jar`、`intellij.libraries.jediterm.ui.jar` 和 `pty4j`，用 `find` 搜 ghostty 一个文件都没有。

![](/images/libghostty/fig1-compare.png)

再从 JetBrains 的发布接口取了 2026.3 的 EAP 包，1510 MB，挂在磁盘镜像里不用装就能看。多出来的东西是这个：

```
Contents/plugins/terminal/libghostty-vt/macos-aarch64/libghostty-vt.dylib
-rw-r--r--  1.9M  Oct 23 13:10
Mach-O 64-bit dynamically linked shared library arm64
```

1.9 MB 的原生动态库，走的是 plugins 目录下的 terminal 插件，不在主 lib 里。有点意外的是 JediTerm 那两个 jar 在 2026.3 里仍然保留着，跟官方说的「可以关掉回退」对得上。

库里导出了 205 个 C 符号，全带 `ghostty_` 前缀。按模块粗分：terminal 31 个、render 22 个、key 21 个、mouse 17 个、kitty 16 个，往后是 snapshot、color、sgr、selection、search、osc 这些。

dylib 里面还内嵌了一份完整的 ABI 类型描述 JSON，开头写着 `"library_version":"0.1.0-dev"`。

![](/images/libghostty/fig2-symbols.png)

然后我把这个 dylib 从 IDEA 里单独拿出来，写了个几十行的 C 程序 dlopen 进去，调 `ghostty_build_info`：

```
VERSION_STRING  rc=0 len=9  "0.1.0-dev"
VERSION_MAJOR/MINOR/PATCH  0.1.0
SIMD=on  KITTY_GRAPHICS=on
```

跑通了。这一点比它换没换内核更值得说。JetBrains 打包进去的是一个标准 C ABI 库，任何程序都能链，不依赖 JVM，不依赖 IntelliJ 平台。

![](/images/libghostty/fig3-probe.png)

官方说的回退开关我也顺手核了，从 terminal.jar 里解出来，`terminal.use.ghostty.emulator` 这个 key 确实在 `TerminalEmulatorType` 和 plugin.xml 里。位置是 Help → Find Action → Registry。

## 这事跟写 Kotlin 和 Android 的人有什么关系

如果你的日常是 IDEA 里开一个终端跑 Claude Code，然后旁边开着 gradlew 和 adb，那这次换内核动的是你每天踩的那块地板。

三个能直接感觉到的：Shift+Enter 不再把半句话发出去；切 Dark 模式终端里的工具跟着变；那些以前显示成乱码的控制字符不再污染输出。

还有一批已经能解析、但 IDE 这边还没接的能力，官方自己列了：OSC 52 剪贴板、桌面通知、Conemu 进度条、Kitty graphics、带颜色和样式的下划线。意思是解析不再是瓶颈，剩下的都在 IDE 侧。

最实用的一条是桌面通知，终端里的程序跑完任务能弹通知，比如 Claude 处理完一个 prompt 在等你。这条现在还没通，官方说会来得比 JediTerm 时代快。

## 我的一点看法

把 13 年的自研终端换成别人家的库，这个决定本身比技术细节更值得琢磨。

JetBrains 有能力继续改进 JediTerm，他们的选择是认账。现代终端协议这一层已经不是顺便能补上的东西，自己从零追不如直接站到一个专门做这件事的库上。

而且 libghostty 的定位本来就是给别人用。Ghostty 官方文档里写得很明确，架构上的目标就是让其他终端项目能基于共享核心构建，专注上层功能和界面，不用重新实现终端模拟。

JetBrains 在博客末尾挨个谢了 Mitchell Hashimoto 和 Tim Culverhouse，说这几个月里 libghostty 也补上了 JediTerm 用户依赖的那些能力，才让这次切换做得成。

这话反过来读也成立。一个被当成业余项目的库，为了接住一个大客户，把对方十几年的需求补进去了。

你现在的日常终端是 IDE 内置那个，还是单独开一个 Ghostty、iTerm、Warp？如果 IDE 的终端真能跟上独立终端的功能，你还会专门再开一个窗口吗？

## 参考资料

- JetBrains 官方博客：The Terminal in JetBrains IDEs Now Runs on libghostty（2026-10-27）
- Ghostty 官方文档 About：https://ghostty.org/docs/about
- libghostty 生态列表：https://github.com/Uzaaft/awesome-libghostty
- libghostty 的 build_info 头文件：https://github.com/ghostty-org/ghostty/blob/main/include/ghostty/vt/build_info.h
