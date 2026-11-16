---
title: "项目"
url: "/projects/"
summary: "我维护的开源项目，以及它们各自的定位"
ShowToc: false
hidemeta: true
comments: false
---

这里是我长期维护的几个开源项目。数字取自 pub.dev 和 GitHub 的公开接口，写这篇时当场查的。

## retrofit.dart

Type-safe REST 客户端生成器，跑在 dio 之上，靠 source_gen 在编译期生成实现。

你写一个抽象类加注解，它给你生成完整的请求代码，路径参数、查询参数、表单、multipart、拦截器、错误处理都不用自己拼。设计上参考了 Chopper 和 Square 的 Retrofit，把 Android 那套声明式接口搬到 Dart。

写好六年多了。从 2019 年 4 月的 0.0.1 到现在的 4.10.0，一共 63 个版本。pub.dev 上 2,116 个 likes，最近 30 天下载 69.9 万次，GitHub 上 1,179 星。依赖只有两个：`dio` 和 `meta`。

```dart
@RestApi()
abstract class UserApi {
  factory UserApi(Dio dio, {String baseUrl}) = _UserApi;

  @GET("/users/{id}")
  Future<User> getUser(@Path("id") int id);
}
```

- pub.dev：<https://pub.dev/packages/retrofit>
- GitHub：<https://github.com/trevorwang/retrofit.dart>
- API 文档：<https://hitrevor.com/retrofit.dart/>

## openai_dart

OpenAI API 的 Dart 客户端，覆盖 REST 接口和 realtime 的 WebSocket / WebRTC 通道，GPT、图像、音频、Responses API 都有类型化封装。

起于 2023 年 10 月，那会儿 GPT 的 Dart 生态还很空。到现在 74 个版本，最新 12.0.0 是 11 月 5 日发的。pub.dev 上 139 个 likes，评分 160 分满分，最近 30 天下载 4.7 万次。依赖是 `http`、`web_socket`、`crypto`、`logging`、`meta`、`http_parser`，没有重的东西。

这个项目有个反差值得说：GitHub 上只有 9 颗星，pub.dev 月下载却有四万七。看仓库的人少，用包的人不少，说明大部分使用者不看源码，直接把它当依赖装进项目。这也是我一直在维护它的原因，工具本身不该需要被研究。

- pub.dev：<https://pub.dev/packages/openai_dart>
- GitHub：<https://github.com/trevorwang/openai_dart>

## chatgpt_book_code

掘金小册《Flutter 实战：ChatGPT 客户端开发》的配套代码。33 颗星，2023 年 11 月最后一次提交。

这份代码是教程的一部分，讲怎么从零搭一个 Flutter 的 LLM 客户端：流式响应、会话管理、上下文裁剪这些。它不跟着 OpenAI 的 API 演进，现在拿去看思路可以，直接跑得自己补更新。

留在那是因为教程还在，代码还得对得上。

- GitHub：<https://github.com/trevorwang/chatgpt_book_code>
