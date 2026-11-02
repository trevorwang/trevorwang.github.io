---
title: "Google 把每个 App 变成了 MCP server"
date: 2026-11-02T10:50:00+08:00
draft: false
categories: ["公众号"]
tags: ["Android", "MCP", "AI Agent"]
summary: "Android 16 上每个 App 都能像 MCP server 一样对外声明函数。我在模拟器上隔着界面发出一条消息，全程没碰屏幕；也翻出了那道 internal|privileged 的权限门。"
cover:
    image: "/images/appfunctions/cover.png"
    alt: "Google 把每个 App 变成了 MCP server"
    relative: false
---

MCP 这一年解决了服务端的事：agent 怎么发现工具、怎么调、怎么拿回结构化结果。手机上一直是空白。你的 App 里有一堆能力，搜索、发消息、加日程，agent 一个都碰不到，只能靠截图猜坐标，点一下看一眼。

今年二月，Google 在 Android 官方博客提了一个东西叫 AppFunctions。上个月二十一号，它的 SDK 发了 1.0.0-beta01。我把它在模拟器上跑通了，包括让「agent」隔着我 App 的界面，直接发出去一条消息。

## 官方的说法很直白

那篇博客是 Android 开发者产品副总裁 Matthew McCullough 写的，原文用了这么一句：AppFunctions 让 App 像后端通过 MCP cloud server 声明能力那样，把能力声明出来，只不过执行发生在设备本地。

同一段里还提了 WebMCP，说这俩是一路：一个在浏览器，一个在系统。

所以定位很清楚，它不是又一个自定义 intent 协议，是有意照着 MCP 的形状做的。10 月那次 beta 更新里有个细节能佐证：他们把元数据字段从 `description` 改名成了 `instructions`，理由写的是「对齐 MCP 命名惯例」。

## 一条 AppFunction 要走四步

第一步，在你的 App 里写一个服务类，继承 `AppFunctionService`，方法上打 `@AppFunctionDeclaration`，KDoc 会被当成给模型的说明。第二步，构建时注解处理器生成一个 XML，把参数、类型、枚举约束、返回结构写进去。

第三步，这个 XML 打进 APK 的 assets 目录，Android 系统读它，把函数注册进一张全局表。

第四步，有权限的调用方拿到 `AppFunctionManager`，先查这个函数开没开，再调。

我特意去 APK 里翻了一遍，这个 XML 确实在。样例工程的 `assets/chat_app_function_service.xml` 二十二 KB，六个函数，参数描述全是从 KDoc 抽的。

![一条 AppFunction 从声明到执行要走四步](/images/appfunctions/fig1.png)

## 本机实测：从编译到真的发出一条消息

样本是官方仓库 `android/appfunctions` 里的 ChatApp，Compose 加 Hilt，编译出十四点八 MB 的 debug 包。装进模拟器，一打开就注册了六个函数：搜索联系人、发消息、拨号、搜消息、查发送详情、换壁纸。

设备那边列出来是这样，同一台模拟器上还有系统自己的人：

- `com.example.chatapp` 六个，全是这个聊天 App 的业务
- `com.android.settings` 六个，全叫 `getXxxDeviceState`，读电池、存储、通知这些状态
- `com.google.android.permissioncontroller` 一个，读权限状态
- `com.google.android.apps.wellbeing` 一个

系统 App 自己在用这套东西，这点比文档更有说服力。

## 调用一次，看它返回什么

我用 ADB 直接调，绕开界面，模拟一个 agent 的行为。先按函数说明里写的「必须先搜联系人拿到 endpointValue」，搜一个人：

```
androidAppfunctionsReturnValue:
  contactDisplayName: Alice Smith
  endpointValue: 1
  contactType: INDIVIDUAL
  endpointDisplayName: alice@example.com
```

拿到 `1` 之后，第二步发消息，参数就是这个 id 加正文。返回一句 `Message sent to: Alice Smith.`。然后我打开 App 看了一眼，那条「这条是 agent 通过 AppFunction 发出来的」就躺在 Alice 的会话里。

全程没有碰过一次界面，没有截屏，没有点坐标。

![隔着界面发一条消息：ADB 两步调用，右边是 App 里那一秒](/images/appfunctions/fig3.png)

## 最该记住的一条：谁能调

这是我最想让你注意的地方。

要让别的应用发现并执行你的 AppFunction，调用方必须持有 `EXECUTE_APP_FUNCTIONS`。我在模拟器上把这个权限的定义翻出来了：

> Permission [android.permission.EXECUTE_APP_FUNCTIONS]
> sourcePackage=android
> prot=internal|privileged

internal 加 privileged，意思是普通 App 拿不到，只有系统签名或者特权白名单里的应用才有。现阶段就是 Gemini 这类系统助手。

所以外面那两种声音都能对上：说「安卓 App 终于能跟 AI agent 对话了」没错，说「这不等于开放 agent 访问」也没错。函数是公开声明的，门是锁着的。

![模拟器上的函数注册表，和那道 internal|privileged 的权限门](/images/appfunctions/fig2.png)

## 它已经在往政策里走

搜索结果的另一条线索是 AEP，Google Play 的 Apps Experience Program。

它在里面有一条明确的准则叫 Android MCP，要求参与的应用必须用 AppFunctions 实现至少一个相关场景，通过必测项。适用范围写得很具体：消息、笔记、日历、任务、健康、购物。执行时间是 2027 年 3 月 1 日，或者自助注册开放后三个月，取较晚的那个。

也就是说，今天它是实验性 API，明年它是 Play 的一条合规项。这个节奏值得提前看一眼。

![从博客到 Play 合规项的八个月，以及今天能验和不能验的边界](/images/appfunctions/fig4.png)

## 还有一条零代码的路线，得放在一起看

同一篇博客里，Google 还放了另一条路：UI 自动化。

意思是平台自己去操作你的界面，开发者的工作量是零。用户在 Galaxy S26 和部分 Pixel 10 上长按电源键，就能把一段多步骤任务交给 Gemini，首发只开美国、韩国的外卖、生鲜、网约车几个品类。

用户能看进度，能随时接管，涉及付款这类敏感动作前会被提醒。

这两条路摆在一起看，取舍就很清楚了：AppFunctions 给你控制权，你自己决定暴露什么、怎么描述、失败怎么回；UI 自动化给你覆盖面，但你不参与定义，agent 只能像人一样点你的界面。

官方说 Android 17 会扩大这两块。我的判断是，认真做产品的会把前者当主场，后者当兜底。

## 还没到的地方，说清楚

SDK 是 beta，不是正式版。官方文档标着 experimental preview，接口还会变，十月那次就动了不少。Gemini 侧的集成现在是私有预览，只对受信任的测试方开放。能真正用起来的公开案例是三星相册，在 Galaxy S26 上，很快扩到 One UI 8.5。

还有一条我得如实说：我试着用 ADB 把其中一个函数关掉，验证「用户可以禁用」这个说法，命令跑了三十秒超时，函数状态没变。这条我没验证成功，不应该写进结论。

## 我的判断

这次和以往那些「AI 集成 API」不一样的地方，在于形态。

过去是各家自己定一套 intent，agent 要逐个适配。这次是照 MCP 的模子刻：自描述、可发现、有枚举约束、有必填项、有错误码。对写 App 的人来说，真正的变化不在今年，在于明年那条政策生效时，你的 App 有没有把自己的核心能力说清楚。

一个想清楚了的函数声明，比你写十页帮助文档更能让 agent 用对。

你手上那个 App，如果只允许暴露三个函数给系统助手，你会挑哪三个？

评论区说说，我挑几个有意思的，下篇拆开写。
