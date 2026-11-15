---
title: "DeepSeek 把 App 拆成了 API，同一天 Harness 学会了导入聊天记录"
date: 2026-11-14T17:06:15+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "11 月 13 日这一天，DeepSeek 做了两件方向相反的事。"
cover:
    image: "/images/dsh-api-import/cover.png"
    alt: "DeepSeek 把 App 拆成了 API，同一天 Harness 学会了导入聊天记录"
    relative: false
---

11 月 13 日这一天，DeepSeek 做了两件方向相反的事。

官方开放了「全家桶应用 API」：联网搜索、文件解析、语音，这些原本只存在于 App 里的能力，现在开了参数就能在 API 里用。宣传语是「1:1 复刻出完整的 DeepSeek App」。

同一天晚上，DeepSeek Harness 发了 v0.2.2-rc.3，新增的只有一件事：把 DeepSeek App 里的历史会话导进工作区，接着聊。

一边把能力往外放，一边把数据往里收。这两件事单独看都不算大新闻，放在一起就有点意思了：官方在拆自己的 App，社区在缝合自己的历史。

我在本机把两边都跑了一遍，有些数字跟文档写的不太一样。

![同一天的两个方向](/images/dsh-api-import/fig1.png)

## 先看官方开放了什么

开放出来的东西实际上分两层，一层是参数开关，一层是独立接口。

第一层是「开关」。在原有的 Messages、Responses、Chat Completions 三个接口里，声明对应工具就能用：`deepseek_web_search` 负责联网搜索和深度思考搜索，`deepseek_read_url` 负责读链接，文件输入和模型安全各有一个参数。

关键在于文档的一句原话：搜索和抓取都在服务端完成，客户端不需要执行任何工具。

这句话决定了它的用法。你不是自己写爬虫再接模型，而是发一次请求，中间那几步服务端替你走完。

第二层是「独立接口」。Search、Fetch、OCR、ASR、TTS 五个能力都能单独调，不经过对话模型。走这条路的话，你在自己拼工作流。

两层加起来，官方给的定位是「打开参数就能复刻 App」。我验了一下这句话有多少水分。

## 一次调用，服务端替我搜了三次

我拿 Responses 接口试了一条最简单的请求：问 Kotlin 2.5.0-RC 什么时候发布、有什么特性，让它用三句话回答。

请求体只有三个字段：模型 `deepseek-flash`、工具 `deepseek_web_search`、一句 input。

回来的响应有 6969 字节，走 4.2 秒。里面不是一条回答，是五个内容块，按顺序排：推理、搜索调用、推理、搜索调用、回答。

第一个搜索块里写着 `kind: "deep_search"`，模型自己发起了三条查询，中英文混着来。一条用中文问发布时间和特性，一条用 `site:kotlinlang.org` 把范围限死在官方站点，还有一条直接找 release announcement 公告页。

三条查询带回来 16 个来源，每个都有 url、title、snippet、site_name 和 published_at。我数了一下，里面既有 kotlinlang.org 的官方文档页，也有第三方论坛的转载。

这就是「服务端替你走完」的真实代价：你说一句话，它在后面搜了三次。

![一次请求的内容块与用量](/images/dsh-api-import/fig2.png)

## 计费按工具执行次数算

响应里有个字段值得单独说：`server_tool_use`。这次调用返回的是 tool_search 三次、tool_fetch 零次、tool_image_search 零次。三次搜索，零次抓取，零次以图搜图。

同一份 usage 里还有 `input_tokens: 11799`（其中 4608 是缓存命中）、`output_tokens: 761`（其中 433 是推理 token）。

一万多输入 token 里，绝大部分是那 16 个来源的内容，不是你的问题。

所以官方说的「除 Token 费用外还收取工具调用费用」，收的就是 `server_tool_use` 这个计数。你在代码里写一行工具声明，账单上就多三行搜索费，而你控制不了它搜几次。

## 五个独立接口，我挨个试了

如果你的场景不需要对话，直接用独立接口更省。我把能本机验证的三个都跑了。

TTS 是最干脆的。发一段中文加一个 voice_id，2.09 秒回一段 wav。默认音色 stella 出来 276,524 字节、5.76 秒、24kHz；换 tide 是 261,164 字节、5.44 秒。同一段文字，男声比女声短 0.32 秒。

ASR 接上一步的 wav 做闭环。1.2 秒返回，`duration_ms: 5760`，跟本机 afinfo 量出来的 5.76 秒完全一致。

但转写结果有个坑。我念的是「多字段的 value class 在字节码里就是一个普通类，没有 box-impl」，它转回来是「多字段的Value Class在字节码里就是一个普通类，没有Box应用」。

`box-impl` 被听成了「Box应用」。

这不是 ASR 独有的毛病。任何把英文技术术语丢进中文语音识别的人都会遇到：模型听见了音，猜了一个最像的中文词。你要是靠语音输入写代码注释，这种错会一路带进仓库。

OCR 我拿了一张自己做的封面图试，2.29 秒，481 输入 token 换 131 输出 token。图上的「36.0 字节 / 次」「0 / 2 迁移开关」这种带小数点和斜杠的排版，全都准确读了出来，连中间那个间隔点都没漏。

![三个独立接口的实测数据](/images/dsh-api-import/fig3.png)

## Harness 这边只加了一件事

同一天晚上 dsh v0.2.2-rc.3 发布，commit `38a6a5c`，作者 imccyu 和 winewill。

改动就两条：新增实验性「DeepSeek 会话导入」插件，修复 Windows 桌面版应用内更新失败。

第二条是纯粹的 bug 修复，不用展开。第一条值得说，因为它跟当天放开的 API 是一对：官方 Demo 支持导入历史对话，dsh 把这个能力做成了插件。

这个插件是可选层，官方明确写了 shipped profiles 默认不含它。要手动挂上去：

```yaml
- insert:  # 挂在 profile 的 patch 层末尾
    - id: deepseek-chat-import
      name: '@deepseek-ai/dsh-deepseek-chat-import'
```

用法是点「新建会话」旁边的上传箭头，选一个 `deepseek_data-*.zip`，再选接收会话的工作区。

## 升上去先撞了一堵墙

我先在自己常用的 web profile 上升级。`npm install -g @deepseek-ai/dsh@0.2.2-rc.3`，装完是 0.2.2-rc.3，然后重启。

启动日志直接刷了一屏：

```
disabling profile plugin row "storage-domain": Plugin
@deepseek-ai/dsh-storage-domain@0.2.2-rc.1 is incompatible with dsh 0.2.2-rc.3
```

storage-domain、plugin-market、cua、ui-cua、tool-plugin-market、cua-provider-registry，六个插件全被禁用。原因都一样：profile 里装的是 rc.1，主包是 rc.3，版本对不上。

然后是一串连锁反应：storageDomain 起不来，workspaceRegistry 就 pending，session-controller 跟着 pending，紧接着我关心的那个插件也挂在列表里，等的是 workspaceRegistry。

插件本身没坏，是被上游拖住了。这一幕恰好是「Everything is a Plugin」这个路线的另一面：组合得越细，一处版本不合就串起来倒一片。

![六个插件因版本不符被禁用](/images/dsh-api-import/fig5.png)

## 换一个干净 profile 才跑起来

我没去动他日常那个 profile 的依赖，而是从官方模板重开了一个，命令是 `dsh --profile chatimport --from-default-profile web --port 3081`，换个端口免得跟原来的撞车。

新 profile 只有 `@deepseek-ai/dsh-base` 和 `@deepseek-ai/dsh-web-app` 两个 bundle，干干净净。加上那段 insert 再启动，日志里没有一行 disabling，也没有 pending。

界面上多了一个按钮。我用无障碍树把页面上所有按钮列出来，第二个就叫 Import DeepSeek chats，38 像素见方，紧挨着新建会话。

插件确实活了。问题变成：拿什么文件喂它。

## 导入实测：三个、两个、一个被跳过

真实的导出包我手上没有，就从插件源码里的 zod schema 反推结构，自己造了一个。

一个合法的导出 ZIP 只要三个东西：`manifest.json` 里写 `export_format_version: 3`，一个 `chat_session_index.json` 列出会话，每个会话一个 `chat_sessions/{id}.json`。会话文件里是消息树，每条消息带 id、parent_id、role、status 和 fragments。

我造了三个会话：一个普通的问答，一个带联网搜索和多轮思考，一个专门用来测过滤。

第一版里，我把「被内容过滤」做成了消息 status 为 `FINISHED`、fragments 里塞一个 `CONTENT_FILTER`。导入结果对话框直接写了一句 Imported 3 sessions into the smoke-test-connectivity-f0b666d1b2bb workspace。

三个全进来了，包括那个本该被丢掉的。我去翻了转换源码，发现判断只认一个字段：`if (reply?.status === 'CONTENT_FILTER')` 就直接 `continue`，连自增计数都是单独一步。

过滤看的是消息自己的 status，不是 fragment 的类型。我改过来重新造包，第二次对话框写的是 Imported 2 sessions，后面紧跟一句 1 conversation could not be imported。

两个进来，一个跳过。跟文档写的「回答被内容过滤撤回的提问与该回答一起略去，没有其他 turn 的会话不导入」对上了。

顺便验证了另一条：同一个包导两次，会话会重复。我工作区的侧边栏里现在是五个会话，其中 Kotlin 那条和 Flutter 那条各有两个副本，被过滤的那条只有一个。导入器不记录哪些已经导过。

![导入漏斗与副本](/images/dsh-api-import/fig4.png)

## 导入是纯本地写入，继续聊才碰模型

打开导入进来的会话，历史是全的：我造的提问、回答、时间戳，渲染成 11/10 17:00，底部写着 1 turns 1 steps。

然后我在这个会话里直接发了一句新的提问，想看它能不能接着聊。结果是红色的一行 This turn failed，说 deepseek-official 这条 provider 路由没有 API key，让我通过 credentials 服务写入，或者在启动环境里 export 一个，末尾挂着一个 MISSING_CREDENTIAL 的代码。

新 profile 没配凭据，这是我的环境问题。但这个报错本身说明了一件事：导入这一步从头到尾没有请求过模型，不然它会先卡在这里。

界面上那行红色的「Injected context」也在说同一件事：历史已经被塞进上下文，只是没人来读它。截图里那次失败的 turn，底部统计变成了 2 turns 2 steps。

## 两条路，各自解决什么

把两边并起来看，它们其实在回答两个不同的问题。

全家桶 API 解决的是「怎么做出一个像 DeepSeek App 的东西」。它把搜索、解析、语音这些脏活包起来卖，你付 token 费加工具调用费，换来不用自己搭检索和解析。

会话导入解决的是「怎么不丢掉自己已经聊过的东西」。它跟模型无关，纯本地读写，把一个 ZIP 拆成会话文件写进工作区。

在我这边，前者四秒能出结果，后者一秒就能把三个会话摆进侧边栏。但前者需要你信任它的服务端替你搜了三次，后者要接受它会把同一个包导入五遍也照样写进去。

同一天发布的这两个东西，一个往外放能力，一个往里收历史。真要挑一个先动手，看你缺的是能力还是记性。
