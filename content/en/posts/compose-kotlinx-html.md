---
title: "Jake's Approach: Run Compose on the Server, Push Only HTML"
date: 2026-11-12T08:10:00+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin", "Compose Multiplatform", "Ktor"]
summary: "Jake Wharton runs Compose on the server with Molecule, renders HTML with kotlinx.html, pushes it over SSE, and merges it with Idiomorph. I rebuilt it and found that whether your text input survives comes down to three defaults."
cover:
    image: "/images/compose-kotlinx-html/cover.png"
    alt: "Jake's Approach: Run Compose on the Server, Push Only HTML"
    relative: false
---

Compose for HTML is still on the official roadmap as isomorphic rendering.

Jake Wharton did not want to wait. In August 2026 he published "Compose & kotlinx.html", describing a setup he assembled at home with Molecule: run Compose on the server, produce an HTML string, push it over SSE, and let Idiomorph merge it into the DOM on the client.

One line of his stuck with me. Client frameworks can fix having stale content, but "you can also solve a leaking house by moving to a different house." It is a sharp remark, and it lands: replacing the whole rendering stack usually costs far more than fixing the one pipe.

I rebuilt it to see how far this homemade approach goes. Along the way I checked one thing: on the same push, with one line of client code changed, does the text in the input box survive.

The conclusion first: the survival rate goes from 0% to 100%, and the entire difference is three default values.

## His pipeline is four stages

The data source is a StateFlow. Molecule runs a `@Composable` function as a state machine, kotlinx.html renders the result into a string, SSE pushes it to the browser, and Idiomorph merges it on the client.

The key part is that the server does not distinguish between "first render" and "subsequent push." The same `userList(users)` function is called for both the initial page and incremental updates.

![The pipeline and version stack](/images/compose-kotlinx-html/fig1.png)

I built mine on Kotlin 2.4.21, Ktor 3.6.1, kotlinx-html 0.12.0, Molecule 2.2.0, Compose runtime 1.12.2, running on JVM 21.

## One function, two call sites

This is the most comfortable part of the whole thing.

What you save is not just the template engine.

Initial page: `appendHTML().div { userList(users) }`.

Push: `moleculeFlow(Immediate) { val users by Store.users.collectAsState(); fragment(users) }`.

Both sides call the same function. No template, no JSON-to-view mapping layer, no "the frontend has to implement this again" problem.

One change, both places take effect.

## Who owns the text in the input box

This is what I most wanted to verify.

The page has one input box. It is not in the StateFlow, it is purely browser state. I typed a sentence into it, then triggered a push, and watched to see whether it was still alive.

The first attempt used `content.innerHTML = html`. The text was gone. The reason is simple: the whole content area is rebuilt, the input box is new, the value is empty, and the caret position and any uncommitted IME candidates go with it. That is the classic cost of overwrite-style updates, and the first problem every frontend framework has to solve.

Switching to Idiomorph, I first used `morphStyle: 'innerHTML'`. The page did not move at all. No error. It just did not move.

Changing it to `outerHTML` still did not preserve it. It took a while to find that there is an `ignoreActiveValue` option, and it is off by default.

![The same push, two different outcomes](/images/compose-kotlinx-html/fig2.png)

With it on, the text survived. But only in one case.

## It protects focus, not every field

I ran each combination three times. The results were blunt.

With innerHTML, the field is cleared three out of three times regardless of where focus is. Morph without `ignoreActiveValue` clears it three out of three. With the option on but focus already moved to a button, still three out of three cleared.

Only one case survives: focus sitting inside that input box.

![Four combinations, three runs each](/images/compose-kotlinx-html/fig3.png)

The reasoning is not hard: Idiomorph protects the currently active element. Other inputs get overwritten by the server HTML, and that `<input>` on the server has no `value` attribute.

In other words, this setup defaults to "form state belongs to the server," and only yields on the one cell the user is currently typing into.

Jake's original text says "form fields that haven't changed are preserved." That holds in the "user is typing right now" scenario. Outside it, it does not.

## Connection count equals render count

I added a counter on the server to see how many times a composition ran.

Idle it is 0 — nothing renders if nobody subscribes. Open ten persistent SSE connections and it becomes 10. Change state once and it becomes 20. Those three numbers are the part of this experiment I most want people to see.

What that means: every data change requires the server to run the composition function once per connection. Ten connections, ten runs. A hundred connections, a hundred runs.

![Render count with 10 connections](/images/compose-kotlinx-html/fig4.png)

A single connection receives a 1,903-byte payload, about 42% of the full page, plus 97 bytes per extra user row. The whole block is pushed with no diffing; diffing is delegated entirely to the browser.

Memory showed no measurable difference. The JVM baseline sits at 147 MB, and going to fifty connections only reached 149 MB, which buries the overhead. Real load testing would have to look at per-connection thread stacks and GC pressure.

## Latency has no story here

Locally end-to-end, from clicking the button to the DOM being updated: inner mode ranged 30 to 107 ms, morph mode 18 to 138 ms.

The two ranges overlap. Neither is clearly faster.

That makes sense: both paths go through SSE, both repaint the DOM once, and the only difference is the merge strategy.

So do not expect this to buy you performance. It buys something else.

## What he calls productionization is mostly "don't do this"

Jake wrote three points at the end. Here is how I read them.

One long connection, not one per content block. The pushed payload needs an envelope saying which part of the page this update touches. Multiple blocks sharing one connection lets the server render only once.

Use SSE, not WebSocket. SSE can ride HTTP/2 multiplexing, event names and payload are naturally separate, which makes a convenient envelope. And when the connection drops the browser reconnects on its own, so you do not write backoff retry logic.

The third is the vaguest and the most important: whether the composition function runs inside the connection or outside it. The former means one per user, the latter means one shared by everyone. The former is simpler, the latter is cheaper.

My experiment was the former, which is why ten connections meant ten renders. Doing it his way means first deciding whose state it is.

## Four traps, three of them silent

First, the Molecule 2.x API changed. `launchMolecule` is now an extension on `CoroutineScope` returning a `StateFlow`; to get a `Flow` you switch to `moleculeFlow(mode) { }`. Code from older docs will not compile, which is at least friendly.

Second, Ktor's SSE requires an explicit `install(SSE)`. Writing only `sse("/path") {}` compiles, then throws `Application plugin SSE is not installed` at startup, buried among dozens of lines of boot log.

Third, Idiomorph's `morphStyle` fails silently when wrong. If the fragment carries its own outer div you must use `outerHTML`. Write `innerHTML` and the page freezes. You assume the push never arrived, but it did.

Fourth, `ignoreActiveValue` defaults to off. Without it, morph and innerHTML are indistinguishable in this scenario.

![The four traps](/images/compose-kotlinx-html/fig5.png)

Three silent failures cost me an afternoon.

## My verdict

The approach works.

But the conditions are narrow.

Low-traffic internal pages, a few dozen users, no need for a frontend framework, written end to end in Kotlin. It is far less trouble than pulling in React, and what the server returns is readable HTML that still shows content when the network is down. For internal tools the payoff is real.

But the moment you hit "many users online, state changing frequently," the N-connections-N-renders problem becomes a real defect. Jake himself warns about it at the end: multiplex, switch to SSE, decide whether composition runs inside or outside the connection.

He also says something more important: this is "a starting point for productionization," not the finish line.

What I took from those four traps is a more general rule: any default that fails silently deserves a documentation check before you start working. This stack has three. Other stacks have their own. Molecule's entry point changed without an error, Ktor's missing plugin blows up only at startup, and Idiomorph's broken `morphStyle` just leaves the page still. Each one quieter than the last.

If you are building internal tools, it is worth half an hour to copy this setup. If you are building a customer-facing product page, let a frontend framework handle it.
