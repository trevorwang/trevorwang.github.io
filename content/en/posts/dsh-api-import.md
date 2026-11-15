---
title: "DeepSeek Opened the App's APIs. Harness Learned to Import History."
date: 2026-11-14T17:06:15+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent", "DeepSeek Harness", "API"]
summary: "DeepSeek opened the App's toolkit as API parameters: search, file parsing, speech. One sentence from me triggered three server-side searches, billed per tool call. Same night, dsh 0.2.2-rc.3 added chat import, and upgrading a profile disabled six plugins."
cover:
    image: "/images/dsh-api-import/cover.png"
    alt: "DeepSeek Opened the App's APIs. Harness Learned to Import History."
    relative: false
---

On November 13, DeepSeek did two things that point in opposite directions.

The official release opened up the "whole-suite app API." Web search, file parsing, speech — capabilities that until then existed only inside the App — now work from the API by declaring a parameter. The pitch line is "reproduce the complete DeepSeek App, 1:1."

That same evening, DeepSeek Harness shipped v0.2.2-rc.3. It added exactly one thing: import the conversation history sitting in the DeepSeek App into your workspace and keep talking.

One side pushes capability out. The other pulls data in. Neither is big news alone; together they say something. The vendor is disassembling its own App while the community stitches its own history back together.

I ran both on my machine. Some numbers don't match the docs.

![Two directions on the same day](/images/dsh-api-import/fig1.png)

## First, What They Actually Opened

What got opened is really two layers: parameter switches, and standalone endpoints.

Layer one is switches. In the existing Messages, Responses and Chat Completions APIs, declare the tool and it works: `deepseek_web_search` handles web search and deep-thinking search, `deepseek_read_url` reads a link, and file input and model safety each get a parameter.

The key is one line of documentation: search and fetching both happen server-side, the client executes no tools.

That line decides how you use it. You don't write your own scraper and wire it to the model; you send one request and the server walks the middle steps for you.

Layer two is standalone endpoints. Search, Fetch, OCR, ASR and TTS can each be called on their own, without going through a chat model. Take that route and you are the one assembling the workflow.

Together, the official positioning is "turn the parameters on and you have the App." I checked how much of that holds up.

## One Call, Three Searches on the Server

I tried the simplest possible request against the Responses API: ask when Kotlin 2.5.0-RC ships and what's in it, answer in three sentences.

The request body had three fields: model `deepseek-flash`, tool `deepseek_web_search`, one line of input.

The response came back at 6,969 bytes in 4.2 seconds. Not one answer — five content blocks in sequence: reasoning, search call, reasoning, search call, answer.

The first search block says `kind: "deep_search"`. The model issued three queries on its own, mixing Chinese and English. One asked about the release date and features in Chinese, one used `site:kotlinlang.org` to pin results to the official site, and one went straight for the release announcement page.

Those three queries brought back 16 sources, each with url, title, snippet, site_name and published_at. I counted: official kotlinlang.org doc pages sitting alongside reposts on third-party forums.

That's the real cost of "the server walks it for you." You say one sentence; it searches three times behind your back.

![Content blocks and usage from a single request](/images/dsh-api-import/fig2.png)

## You Pay Per Tool Execution

One field deserves its own section: `server_tool_use`. This call returned tool_search 3, tool_fetch 0, tool_image_search 0. Three searches, no fetches, no reverse image search.

The same usage block carries `input_tokens: 11799` (4,608 of them cache hits) and `output_tokens: 761` (433 of them reasoning tokens).

Of those eleven thousand input tokens, the overwhelming majority is the content of those 16 sources, not your question.

So when the docs say "tool call fees are charged on top of token fees," the thing being counted is `server_tool_use`. You write one line of tool declaration in your code, the bill grows three lines of search fees, and you don't control how many times it searches.

## Five Standalone Endpoints, Tried One by One

If your use case doesn't need conversation, calling the standalone endpoints is cheaper. I ran the three I could verify locally.

TTS was the cleanest. Send a Chinese paragraph plus a voice_id, get a wav back in 2.09 seconds. The default voice stella returns 276,524 bytes, 5.76 seconds, 24kHz; switching to tide gives 261,164 bytes, 5.44 seconds. Same text, male voice 0.32 seconds shorter than female.

ASR closed the loop on that wav. 1.2 seconds to return, `duration_ms: 5760`, matching the 5.76 seconds afinfo measured locally.

The transcript has a trap, though. What I said was "多字段的 value class 在字节码里就是一个普通类，没有 box-impl" — a multi-field value class is just an ordinary class in bytecode, no box-impl. What came back was "多字段的Value Class在字节码里就是一个普通类，没有Box应用."

`box-impl` came back as "Box应用" — Box app.

This isn't an ASR-specific defect. Anyone dropping English technical terms into Chinese speech recognition hits it: the model hears the sound and guesses the closest Chinese word. Write code comments by voice and that error walks straight into the repo.

For OCR I used a cover image I'd made myself: 2.29 seconds, 481 input tokens for 131 output tokens. Layout like "36.0 字节 / 次" (36.0 bytes per call) and "0 / 2 迁移开关" (0 of 2 migration switches) — decimal points, slashes and all — came back accurate, right down to the middle-dot separator.

![Measured numbers for the three standalone endpoints](/images/dsh-api-import/fig3.png)

## Harness Added Exactly One Thing

That same evening dsh v0.2.2-rc.3 landed, commit `38a6a5c`, by imccyu and winewill.

Two changes: a new experimental "DeepSeek chat import" plugin, and a fix for in-app updates failing on the Windows desktop build.

The second is a plain bug fix, nothing to unpack. The first is worth discussing, because it pairs with the API released that day: the official demo supports importing past conversations, and dsh turned that into a plugin.

The plugin is an optional layer — the release notes state plainly that shipped profiles don't include it. You mount it by hand:

```yaml
- insert:  # appended to the end of the profile's patch layer
    - id: deepseek-chat-import
      name: '@deepseek-ai/dsh-deepseek-chat-import'
```

Usage: click the upload arrow next to "New session," pick a `deepseek_data-*.zip`, then pick the workspace that should receive the conversations.

## The Upgrade Hit a Wall

I upgraded my everyday web profile first. `npm install -g @deepseek-ai/dsh@0.2.2-rc.3`, it installed 0.2.2-rc.3, I restarted.

The startup log flooded a screen:

```
disabling profile plugin row "storage-domain": Plugin
@deepseek-ai/dsh-storage-domain@0.2.2-rc.1 is incompatible with dsh 0.2.2-rc.3
```

storage-domain, plugin-market, cua, ui-cua, tool-plugin-market, cua-provider-registry — all six disabled. Same reason every time: the profile has rc.1, the main package is rc.3, the versions don't line up.

Then the chain reaction: storageDomain can't start, so workspaceRegistry goes pending, session-controller follows it into pending, and the plugin I actually care about sits in the list waiting on workspaceRegistry.

Nothing was broken about the plugin. It was held up by upstream. This is the other face of the "Everything is a Plugin" philosophy: the finer the composition, the further one version mismatch cascades through the whole stack.

![Six plugins disabled for version mismatch](/images/dsh-api-import/fig5.png)

## A Clean Profile and It Runs

Rather than touch the dependencies of the profile I use daily, I spun up a fresh one from the official template: `dsh --profile chatimport --from-default-profile web --port 3081`, a different port so it wouldn't collide with the original.

The new profile carries just two bundles, `@deepseek-ai/dsh-base` and `@deepseek-ai/dsh-web-app`. Clean. Add that insert block, restart, and the log shows no disabling lines and nothing pending.

The UI gained a button. I dumped every button on the page through the accessibility tree; the second one is called Import DeepSeek chats, 38 pixels square, right next to New session.

So the plugin is alive. The question becomes what to feed it.

## Import Test: Three, Two, One Skipped

I don't have a real export package, so I reverse-engineered the structure from the zod schema in the plugin source and built one.

A valid export ZIP needs three things: `manifest.json` with `export_format_version: 3`, a `chat_session_index.json` listing sessions, and one `chat_sessions/{id}.json` per session. The session file holds a message tree, each message carrying id, parent_id, role, status and fragments.

I built three sessions: a plain Q&A, one with web search and multi-round reasoning, and one designed to test filtering.

In the first version I made "content filtered" mean message status `FINISHED` with a `CONTENT_FILTER` fragment stuffed in. The result dialog said, flatly: Imported 3 sessions into the smoke-test-connectivity-f0b666d1b2bb workspace.

All three came through, including the one that should have been dropped. I went into the conversion source and found the check reads exactly one field: `if (reply?.status === 'CONTENT_FILTER')` then `continue`, with the counter increment as its own separate step.

The filter looks at the message's own status, not the fragment type. I fixed that and rebuilt the package; the second dialog said Imported 2 sessions, followed by 1 conversation could not be imported.

Two in, one skipped. That matches the docs: "a question whose answer was withdrawn by content filtering is omitted along with the answer, and sessions with no other turn are not imported."

Which confirmed something else: import the same package twice and the sessions duplicate. My workspace sidebar now shows five sessions, with the Kotlin one and the Flutter one each having two copies, and the filtered one still single. The importer keeps no record of what it has already imported.

![The import funnel and the duplicates](/images/dsh-api-import/fig4.png)

## Import Writes Locally; Only Chatting Touches the Model

Opening an imported session, the history is complete: my questions, my answers, the timestamps, rendered as 11/10 17:00, with 1 turns 1 steps at the bottom.

Then I sent a new question straight into that session to see whether it could keep talking. I got a red line: This turn failed, saying the deepseek-official provider route has no API key, to write one through the credentials service or export one in the startup environment, with a MISSING_CREDENTIAL code at the end.

The new profile has no credentials configured — that's my environment. But the error itself proves something: import never called the model at any point, or it would have stalled here first.

The red "Injected context" line in the UI says the same thing: the history is already in context, there's just nobody reading it. In the screenshot, that failed turn pushed the bottom stats to 2 turns 2 steps.

## Two Roads, Two Different Problems

Put the two side by side and they answer different questions.

The whole-suite API answers "how do I build something that behaves like the DeepSeek App." It wraps the dirty work — search, parsing, speech — and sells it to you: token fees plus tool fees, in exchange for not building your own retrieval and parsing.

Chat import answers "how do I stop losing what I've already talked through." It has nothing to do with the model — pure local read and write, splitting a ZIP into session files and writing them into a workspace.

On my machine the first produces a result in four seconds, the second puts three sessions into the sidebar in one. But the first asks you to trust that its server searched three times on your behalf, and the second asks you to accept that importing the same package five times will write five copies.

Two things shipped the same day: one pushing capability out, one pulling history in. If you have to pick one to start with, it comes down to whether you're short on capability or short on memory.
