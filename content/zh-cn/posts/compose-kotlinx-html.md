---
title: "Jake 的办法：服务端跑 Compose，只推 HTML"
date: 2026-11-12T08:10:00+08:00
draft: false
categories: ["公众号"]
tags: ['Kotlin', 'Compose Multiplatform', 'Ktor']
summary: "Jake Wharton 用 Molecule 在服务端跑 Compose、用 kotlinx.html 出 HTML、用 SSE 推送、用 Idiomorph 合并。我照着搭了一遍，发现输入框里的字保不保得住，全在三个默认值上。"
cover:
    image: "/images/compose-kotlinx-html/cover.png"
    alt: "Jake 的办法：服务端跑 Compose，只推 HTML"
    relative: false
---

Compose for HTML 这件事，官方还在做同构渲染的路上。

Jake Wharton 不想等。2026 年 8 月他写了一篇《Compose & kotlinx.html》，说自己用 Molecule 在家里攒了一套：服务端跑 Compose，产出 HTML 字符串，用 SSE 推给浏览器，客户端拿 Idiomorph 合并进 DOM。

他那句话我记了很久。说客户端框架能解决内容是 stale，但「房子漏水，你也可以通过换房子来解决」。这话刻薄，但说到了点子上：换一整套渲染栈，代价往往比修那根管子大得多。

我照着搭了一遍，想看看这套土办法能走多远。顺便验了一件事：同一次推送，客户端换一行处理代码，输入框里的字活不活得下来。

结论先放这儿：保住率从 0% 到 100%，差别全在三个默认值上。

## 他的链路只有四段

数据源是 StateFlow，Molecule 把 `@Composable` 函数当状态机跑，kotlinx.html 把结果渲染成字符串，SSE 推给浏览器，Idiomorph 在客户端合并。

关键在于，服务端不区分「首次渲染」和「后续推送」。同一个 `userList(users)` 函数，首页和增量更新都调它。

![链路与版本栈](/images/compose-kotlinx-html/fig1.png)

我搭的时候用的是 Kotlin 2.4.21、Ktor 3.6.1、kotlinx-html 0.12.0、Molecule 2.2.0，Compose runtime 1.12.2，跑在 JVM 21 上。

## 一个函数，两处调用

这是整套东西里最舒服的地方。

省掉的不只是模板引擎。

首屏：`appendHTML().div { userList(users) }`。

推送：`moleculeFlow(Immediate) { val users by Store.users.collectAsState(); fragment(users) }`。

两边调的是同一个函数。没有模板，没有 JSON 到视图的映射层，也没有「前端要再实现一遍」的问题。

一处改，两处生效。

## 输入框里的字归谁

这是我最想验的一点。

页面上有一行输入框，它不在 StateFlow 里，纯粹是浏览器自己的状态。我往里敲一句话，然后触发一次推送，看它还活着没有。

第一次用 `content.innerHTML = html`，字没了。原因简单：整个内容区被重建，输入框是新的，值是空的，连光标位置和未提交的输入法候选都没了。这是覆盖式更新最典型的代价，也是所有前端框架最早要解决的那个问题。

换成 Idiomorph，我一开始用的是 `morphStyle: 'innerHTML'`。页面一动不动。不报错，就是不动。

改成 `outerHTML`，还是没保住。查了半天才发现有个 `ignoreActiveValue` 选项，默认关着。

![同样一次推送，两种落地结果](/images/compose-kotlinx-html/fig2.png)

加上之后，保住了。但只在一种情况下保住。

## 保护的是焦点，不是所有字段

我把每种组合各跑三次，结果很干脆。

innerHTML，无论焦点在哪，三次都清空。morph 不加 `ignoreActiveValue`，三次都清空。加了选项但焦点已经移到按钮上，还是三次都清空。

只有一种情况保住：焦点停在那个输入框里。

![四种组合各跑三次](/images/compose-kotlinx-html/fig3.png)

道理不难想：Idiomorph 保护的是当前活跃元素，别的输入框会照服务端 HTML 覆盖，而服务端那个 `<input>` 没有 `value` 属性。

换句话说，这套方案默认「表单状态归服务端管」，只在用户正在敲字的那一格上让一步。

Jake 原文说的是「没有变化的表单字段会保留」。这句话在「用户正在打字」这个场景里成立，超出了就不成立。

## 连接数就是渲染次数

我在服务端加了个计数器，数 composition 跑了几次。

空载时是 0，没人订阅就不渲染。开十条常驻 SSE 连接，变成 10。状态改一次，变成 20。这三行数字是我这次实验里最想让人看见的部分。

也就是说，每次数据变化，服务端要为每条连接各跑一遍组合函数。十条连接十次，一百条连接一百次。

![10 条连接下的渲染次数](/images/compose-kotlinx-html/fig4.png)

单条连接收到的载荷是 1,903 字节，大约占整页的 42%，每多一行用户再加 97 字节。整块推，不做 diff，diff 全交给浏览器。

内存上反而测不出差别。JVM 基线就占 147 MB，加到五十条连接也只涨到 149 MB，把这部分开销盖过去了。真要压测，得从每条连接的线程栈和 GC 压力看。

## 延迟这边没什么戏

本机端到端，从点按钮到 DOM 更新完，inner 模式 30 到 107 毫秒，morph 模式 18 到 138 毫秒。

两组数字重叠，没有谁明显更快。

这也合理：两种方式都要走一遍 SSE，都要重画一次 DOM，差别只在合并策略。

所以别指望用这个换性能。它换的是别的东西。

## 他说的生产化，其实就是别这么干

Jake 在文末写了三条，我照着理解一下。

一条长连接，别每块内容各开一条。推的东西要有个信封，说明这次更新的是页面哪一块。多块内容共用一条连接，服务端才能只渲染一次。

用 SSE，别用 WebSocket。SSE 能走 HTTP/2 的多路复用，事件名和载荷天然分开，正好当那个信封。而且连接断了浏览器会自己重连，不用你写退避重试。

第三条最含糊也最关键：组合函数跑在连接里，还是跑在连接外。前者是每个用户一份，后者是所有人共享一份。前者简单，后者省。

我这次的实验是前者，所以十条连接就是十次渲染。要按他说的改，得先想清楚状态是谁的。

## 四个坑，三个是静默的

第一个，Molecule 2.x 的 API 变了。`launchMolecule` 现在是 `CoroutineScope` 的扩展，返回 `StateFlow`，要拿 `Flow` 得改用 `moleculeFlow(mode) { }`。照老文档写编译不过，这个还算友好。

第二个，Ktor 的 SSE 要显式 `install(SSE)`。只写 `sse("/path") {}` 能编译，启动时抛 `Application plugin SSE is not installed`，那行报错埋在几十行启动日志中间。

第三个，Idiomorph 的 `morphStyle` 写错不报错。片段自带外层 div 时必须是 `outerHTML`，写成 `innerHTML` 页面静止，你以为推送没到，其实到了。

第四个，`ignoreActiveValue` 默认关。不开它，morph 和 innerHTML 在你这个场景里没有区别。

![四个坑](/images/compose-kotlinx-html/fig5.png)

三个静默失败，够我一下午。

## 我的判断

这套方案成立。

但成立的条件很窄。

低流量的后台页面，几十个用户，不需要前端框架，用 Kotlin 一处写完。它比引一套 React 省事得多，而且服务端返回的就是可读的 HTML，断网也能看见内容。对内部工具来说，这套组合的收益是实打实的。

但一到「多人在线、状态高频变化」的场景，N 条连接 N 次渲染这件事就变成硬伤。Jake 自己在文末也提醒了，要 multiplex、要换 SSE、要想清楚组合跑在连接内还是连接外。

他还有一句更重要的话：这只是「生产化的起点」，不是终点。

我倒是从这四条坑里学到一个更通用的判断：凡是「不报错但没效果」的默认值，都值得先查一遍文档再动手。这套方案里有三个，别的地方也有。Molecule 的入口签名变了没报错，Ktor 缺插件在启动时才炸，Idiomorph 的 morphStyle 写错只是页面不动，一个比一个安静。

你要是在写内部工具，值得花半小时照抄一遍。要是在写对外的产品页，还是让前端框架去管吧。
