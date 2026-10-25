---
title: "Genkit Dart 1.0：AI 工具箱配齐了"
date: 2026-10-11T10:17:07+08:00
draft: false
categories: ["公众号"]
tags: ["Flutter"]
summary: "Flutter 里加 AI 功能，过去无非两条路：把模型的 key 塞进 App 直接调，或者用 http 手拼请求。前者等于把钥匙交出去，后者写到第三天你会发现，自己在手搓一个拙劣的框架，重试、流式、工具调用、结构化输出，样样都得补。"
cover:
    image: "/images/genkit-dart/cover.png"
    alt: "Genkit Dart 1.0：AI 工具箱配齐了"
    relative: false
---

## 起点：Dart 一直缺个趁手的 AI 框架

Flutter 里加 AI 功能，过去无非两条路：把模型的 key 塞进 App 直接调，或者用 `http` 手拼请求。前者等于把钥匙交出去，后者写到第三天你会发现，自己在手搓一个拙劣的框架，重试、流式、工具调用、结构化输出，样样都得补。

10 月 8 日 Genkit Dart 发了 1.0：

> Today, we're announcing Genkit Dart 1.0, the first stable, production-ready release of Google's open-source framework for building AI-powered features and agents in Dart.

这篇不追热点，只回答两个问题：1.0 到底做了什么，Dart 开发者能从中拿走什么。

## 一套 API，通吃四家模型

Gemini、Claude、OpenAI，以及所有 OpenAI 兼容的服务，走同一个接口。供应商按插件注册，换模型不动业务代码：

```dart
final ai = Genkit(plugins: [googleAI(), anthropic()]);

final fromGemini = await ai.generate(
  model: googleAI.gemini('gemini-flash-latest'),
  prompt: prompt,
);
final fromClaude = await ai.generate(
  model: anthropic.model('claude-sonnet-5-5'),
  prompt: prompt,
);
```

对国内开发者来说，更实际的是那句"OpenAI 兼容"：DeepSeek 这类服务在 `genkit_openai` 插件里有一等公民句柄，`deepSeek.model('deepseek-flash')` 就是全部配置，不用自己拼 baseUrl 假装成 OpenAI。

## flow：带类型的 AI 函数，还能当服务端

官方对 flow 的定义是 strongly typed, observable functions。我的理解：把 AI 逻辑包成一个输入输出都有类型的函数。

类型用 `schemantic` 在 Dart 里写一次：

```dart
@Schema()
abstract class $RecipeRequest {
  String get dish;
  int get servings;
}
```

服务端拿它约束模型输出，Flutter 端拿同一份类型解析结果。中间那层手写的 `jsonDecode` 加 `as Map`，可以整段删掉：

```dart
final recipeFlow = ai.defineFlow(
  name: 'recipeFlow',
  inputSchema: RecipeRequest.$schema,
  outputSchema: Recipe.$schema,
  fn: (RecipeRequest input, _) async {
    final res = await ai.generate(
      model: deepSeek.model('deepseek-flash'),
      prompt: '给我一份 ${input.servings} 人份的「${input.dish}」的做法。',
      outputSchema: Recipe.$schema,
    );
    return res.output!;   // 这里已经是 Recipe 类型
  },
);
```

flow 还能直接挂成 HTTP 端点，客户端用 `defineRemoteAction` 调它，schema 依然共用：

```dart
final router = GenkitRouter()..addAction(recipeFlow);
final server = await router.serve(port: 8123);
```

线上协议是 `{"data": {...}}` 进、`{"result": {...}}` 出。对 Flutter 开发者来说，这意味着一份类型既当接口文档，又当解析器。

## 三种放置方式，key 摆在哪由你定

这是我认为 1.0 最实在的一处。官方明说 AI 逻辑可以放三个位置：

- 直接写在 Flutter 里，适合做原型和 bring-your-own-key 的场景
- 放在 Dart 服务端，敏感提示词留在后端，Flutter 端用 `defineRemoteAction` 调
- AI 逻辑留在 Flutter，模型请求走一个轻量的 Genkit 后端，由它来保护 key 和做鉴权

官方公告里那句提醒值得抄下来：`never embed private API keys in a published client app`。过去我们在客户端里嵌 key 是被现实逼的，现在有了正规出路。

## 工具、中断、中间件

工具让模型调你的 Dart 函数。真正需要人点头的动作，可以在工具里返回 `.interrupt(...)`，把生成循环挂起，等客户端确认后再恢复：

```dart
fn: (input, ctx) async {
  if (ctx.resumed == null) {
    return .interrupt({'hotel': input.hotelName, 'total': input.totalPrice});
  }
  return .response(confirmation.id);
},
```

审批写在工具内部，模型绕不过去。官方原话是 `Putting the approval check inside the tool guarantees that the model can't bypass it.` 这一点比"让模型守规矩"的提示词工程靠谱。

中间件这边，自动重试、从 `SKILL.md` 动态加载技能、工具审批白名单，都是一行 `use: [...]` 挂到 `generate` 上的事。要写自己的横切逻辑，`defineGenerateMiddleware` 留了口子。

## 提示词与生产可观测

Dotprompt 把模板、模型配置、输入输出 schema 收在 `.prompt` 文件里，从 `prompts/` 目录自动加载，Dart 里当函数调：

```dart
final introPrompt = await ai.prompt('destinationIntro');
final response = await introPrompt({'destination': 'Kyoto'});
```

上线之后，`genkit_otel` 按 OpenTelemetry 的 GenAI 语义约定导出 trace、token 用量和延迟，直接接进你已有的观测后端。

## 还在实验区的两个

有状态 agent 和生成式界面。前者用 `defineAgent` 把模型、工具、系统指令、会话状态收进一次定义，会话靠 session store 跨重启保住。后者是 `genkit_a2ui`：agent 不再只吐文本，而是吐日期选择器、表单、确认卡这类组件，Flutter 端增量渲染成原生 widget。

两个都从 `package:genkit/experimental.dart` 导入，可以提前试，不背稳定版的包袱。

## 开发者实际能拿走什么

第一是少写四类胶水：多厂商适配、结构化输出解析、流式传输、工具调用与重试。这四样以前每接一家模型就要重写一遍，现在都收进框架，代码从"每次都重写"变成"接进来就有"。

第二是两处位置挪对了。key 从客户端搬回后端，这是安全账；schema 从文档变成代码，这是接口契约。做过一遍就知道，这两样省掉的是后面几个月的手工活。

第三是能渐进接入。先把它当一个"能调模型的库"用，需要时再上 flow，最后才考虑 agent，不要求你一次把架构换掉。

我在本机把四个场景都跑了一遍，都是真打模型（DeepSeek）不是纸面推演：

```
[flow]   类型安全输出 → 番茄炒蛋（2 人份）| 6 步 / 15 分钟 / 1812ms
[tool]   finishReason=interrupted → restart 后 stop
[client] 远端返回 → 葱油拌面（1人份） | 5 步 | 2686ms
curl  →  {"data":{...}} 进、{"result":{...}} 出
```

![](/images/genkit-dart/fig-map.png)

## 上手前要过的两道坎

成绩说完说得失。1.0 有个隐形门槛：要求 Dart SDK 3.12 以上。本机是 3.11.1，于是 `dart pub add genkit` 安静地给我装了 0.16.1，不报错、不警告。升完 SDK 还有第二道，代码生成器在 1.0 拆成了独立的 `schemantic_builder`，漏装它 `build_runner` 会打印 `wrote 0 outputs`，同样是零报错。

两条自检命令，抄走就行：

```
dart pub deps --style=compact | grep "^- genkit"   # 看真实拿到的版本
ls lib/*.g.dart                                    # 看代码生成有没有真落盘
```

## 还有没验的

诚实交代：Dev UI（`genkit start`，需要 npm 上的 `genkit-cli`）、中间件、Dotprompt 的运行效果、`genkit_otel` 的导出，以及实验区那两个，我这轮都没在本机跑，上面写的是官方文档口径。

值得一提的是，官方样例就在本地缓存里，`~/.pub-cache/hosted/pub.dev/genkit-1.0.0/example/http_server_example.dart`，流式、鉴权、自建路由都写在里面。动手之前先读它，能省半天。

如果你的 Dart 项目要接第一段 AI 逻辑，你会把它放在 App 里还是服务端？欢迎评论区聊聊。
