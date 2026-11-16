---
title: "Projects"
url: "/en/projects/"
summary: "The open-source projects I maintain, and what each of them is for"
ShowToc: false
hidemeta: true
comments: false
---

These are the open-source projects I keep up over the long run. Every number below came from the pub.dev and GitHub public APIs when I wrote this page.

## retrofit.dart

A type-safe REST client generator built on dio, with implementation generated at compile time by source_gen.

You write an abstract class with annotations and it generates the request code for you: path parameters, query parameters, form and multipart bodies, interceptors, error handling. The design follows Chopper and Square's Retrofit, bringing the declarative interface style from Android over to Dart.

Been at it for six years now. From 0.0.1 in April 2019 to 4.10.0 today, 63 releases in total. On pub.dev it has 2,116 likes and 698,997 downloads in the last 30 days; 1,179 stars on GitHub. Only two dependencies: `dio` and `meta`.

```dart
@RestApi()
abstract class UserApi {
  factory UserApi(Dio dio, {String baseUrl}) = _UserApi;

  @GET("/users/{id}")
  Future<User> getUser(@Path("id") int id);
}
```

- pub.dev: <https://pub.dev/packages/retrofit>
- GitHub: <https://github.com/trevorwang/retrofit.dart>
- API docs: <https://hitrevor.com/retrofit.dart/>

## openai_dart

A Dart client for the OpenAI API, covering both the REST endpoints and the realtime WebSocket and WebRTC channels, with typed wrappers for GPT, image, audio and the Responses API.

It started in October 2023, when the Dart side of the GPT ecosystem was still mostly empty. It is now on 74 releases, with 12.0.0 published on November 5. On pub.dev it has 139 likes, a full 160/160 score, and 47,642 downloads in the last 30 days. Dependencies are `http`, `web_socket`, `crypto`, `logging`, `meta` and `http_parser` — nothing heavy.

There is one contrast worth pointing out: nine stars on GitHub against forty-seven thousand monthly downloads on pub.dev. Few people read the repository, many people use the package. Most consumers never look at the source, they just add it as a dependency. That is exactly why I keep maintaining it. A tool should not require being studied.

- pub.dev: <https://pub.dev/packages/openai_dart>
- GitHub: <https://github.com/trevorwang/openai_dart>

## chatgpt_book_code

Companion code for the Juejin booklet *Flutter in Practice: Building a ChatGPT Client*. 33 stars, last commit November 2023.

This code is part of a tutorial: how to build a Flutter LLM client from scratch, covering streaming responses, conversation management and context trimming. It does not track the evolution of OpenAI's API. You can still read it for the approach, but running it today means updating it yourself.

It stays up because the booklet is still there and the code still has to match it.

- GitHub: <https://github.com/trevorwang/chatgpt_book_code>
