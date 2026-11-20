---
title: "DeepSeek Harness rc.5: The Upgrade Got Stuck on Two Packages"
date: 2026-11-19T06:54:00+08:00
draft: false
categories: ["WeChat"]
tags: ["AI Agent", "DeepSeek"]
summary: "rc.5 changed exactly one thing: DeepSeek models use server-side search. I upgraded from rc.1, and both installs failed on two packages that have nothing to do with rc.5. Deleting them was what got it through."
cover:
    image: "/images/dsh-rc5/cover.png"
    alt: "DeepSeek Harness rc.5: The Upgrade Got Stuck on Two Packages"
    relative: false
---

rc.5's changelog has a single line. I upgraded dsh from rc.1, spent over two hours on it, and got blocked by two packages along the way.

Neither package is an rc.5 problem. They had both stopped in the past.

## What rc.5 changed

The official release notes, one sentence: DeepSeek models use DeepSeek server-side search, and search is not billed per query; other models default to the DeepSeek Search API.

Published 2026-11-18 12:51, filed under experience improvements.

The change itself is worth noticing. Search not being billed per query means the model decides whether to search and how many times, and none of it turns into a separate line on the bill. Tool calls like this used to be exactly where repeated charges piled up.

The two kinds of search are not the same thing. The DeepSeek Search API calls out to a retrieval service, billed per query, one tick for every lookup. Server-side search has the model retrieving inside its server-side tool loop, and from the user's side it isn't counted. rc.5 switches DeepSeek's own models onto the former, the free path, by default, while other models still go through the Search API.

## But latest still points at rc.1

I went to npm to look at the versions first.

The dist-tags on `@deepseek-ai/dsh` look like this: `latest` is `0.2.2-rc.1`, `next` is `0.2.2-rc.5`.

Which means anyone running `npm install -g @deepseek-ai/dsh` still installs that rc.1 from November 4. To get rc.5 you have to write `@next` explicitly, or the full version number.

![On npm the latest and next tags point at different versions](/images/dsh-rc5/fig2.png)

This isn't a typo. The release process separates "pre-release" from "stable"; the rc line stays on next, and latest only moves at a formal release.

## The ledger across four versions

Three versions sit between rc.1 and rc.5, and they differ a lot.

rc.2 was a big one: Git Worktree reviewing changes by branch, commit and file type, SubAgent supporting named and multi-level collaboration, the desktop app able to switch between stable and dev update channels in settings, followed by twenty-odd experience improvements.

A few that affect daily use: scrolling back in a chat auto-loads earlier messages and holds your reading position; pasting an image no longer requires the input box to be focused; after session restore or compaction it says explicitly how the new context replaces the old; API keys became masked and no longer show the full old key. That last one stuck with me — the previous version displayed the old key verbatim in the UI.

rc.3 added just one experimental session import plugin, pulling history from the DeepSeek App or the web version into the workspace, plus a fix for in-app updates on Windows not opening.

rc.4, two fixes: handling when a model returns an abnormal message, and LiveShare on desktop failing to create a share.

rc.5, one: the search line above.

My machine had been sitting on rc.1, because I tried rc.3 on November 13, six plugins got disabled, and I rolled back. This time, going to rc.5, I wanted to see whether that batch of plugins survived.

![Three versions sit between rc.1 and rc.5](/images/dsh-rc5/fig1.png)

## First install, stuck on a /tmp path

With version numbers bumped in the profile's dependencies, the first `pnpm install` failed outright.

The error is `Could not install from "/tmp/dsh-plugin-build/dsh-plugin-profile-ping-0.1.0.tgz" as it does not exist`.

I wrote that plugin. Its dependency pointed at a tarball under `/tmp`, and that directory had been cleared long ago.

At the time I assumed it was just "one plugin won't install back". Only this time did I find the cost is larger: it makes dependency resolution fail for the whole profile, and not one of the twenty-odd packages installs.

The plugin itself isn't broken; its source path isn't reproducible. A plugin installed from `/tmp` can drag down a reinstall of the entire profile.

I pointed the path at the backup copy under `~/.dsh/plugin-archives/`, the 4,162 bytes were still there, and pnpm went straight through.

One detail here is easy to miss. pnpm reports "dependency tree resolution failed", not "this package failed to install". It stops resolving the other packages too, because the premise of the whole tree has been disproved. So the cost of a plugin with a dead source isn't its own absence — it's that every plugin fails to install together.

![Two failed installs side by side](/images/dsh-rc5/fig3.png)

## Second install, stuck on a package that stopped publishing

Path fixed, the second round failed with a different error: `No matching version found for @deepseek-ai/dsh-experimental-client-ui-agent-team@0.2.2-rc.5`.

I looked up that package's publish history on the official registry.

The last publish was `0.2.2-rc.1`, at 2026-11-04 09:09. The registry's `modified` field is frozen at that moment too.

In other words, when rc.2, rc.3, rc.4 and rc.5 went out, this package never moved once.

Not a mirror delay. I checked: npmmirror and registry.npmjs.org agree, it simply wasn't published.

## Config tree diff: eight in, three out

I did the dumb thing. Dumped the full config tree of rc.1 and of rc.5, and compared line by line.

rc.1 is 1,319 lines, rc.5 is 1,355, 36 more. At the plugin id level, rc.5 has eight new ones:

`agent-team`, `tool-agent-team`, `ui-account-notifications`, `mcp-manager`, `office-kit`, `workspace-dependencies`, `skill-office`, `forge-panel`.

Three that rc.1 has and rc.5 doesn't: `account-controller`, `office-to-pdf`, `optional-forge-panel`.

Seeing `agent-team` and `tool-agent-team` in the added list, I understood why that package stopped publishing.

It was absorbed into the core. What used to be provided by the `dsh-experimental-client-ui-agent-team` package is now in the main package directly, as `@deepseek-ai/dsh-agent-team` and `@deepseek-ai/dsh-tool-agent-team`.

The other three are the same reorganization: `office-to-pdf` merged into `office-kit`, `account-controller` renamed to `ui-account-notifications`, `optional-forge-panel` dropped the optional prefix.

## Delete those two packages and the tree doesn't change at all

I made a test profile, installed rc.5, but deleted both agent-team packages entirely from the dependency and bundle lists.

After installing I dumped the config: 218 plugin ids.

Compare that with "rc.5 with both packages installed": also 218, not one more, not one less.

Deleting them loses nothing, because the core already provides what they provide.

![Config tree differences between rc.1 and rc.5](/images/dsh-rc5/fig4.png)Keeping them only adds a version constraint, which blocked the whole upgrade.

## Upgrading isn't the same as having swapped

Core at rc.5, profile reinstalled, I thought I was done.

`lsof` showed the process on port 3080 was still the one running since November 14. It was running code loaded into memory back in the rc.1 era.

`dsh --version` says rc.5; the service itself hasn't changed. These are two separate things.

The reason: a Node process loads modules into memory at startup, and whatever happens to the packages on disk afterwards doesn't touch it. npm installing packages changes files; a process can't change itself.

Kill it and restart, and only then does the new service actually run rc.5. The log shows the sticky-wall plugin ready, memorix's MCP service up, all 218 plugins loaded, not a single disabled record.

![The four steps, in reverse of the order I hit them](/images/dsh-rc5/fig5.png)

One npm detail while I'm here. npm 12 blocked install scripts for five packages on the global install, two of them native modules, koffi and node-pty. I worried the terminal would break, went through the directory, and node-pty ships `prebuilds/darwin-arm64/pty.node`, prebuilt, so it works without running install scripts.

## So how should the upgrade go

Reversing the order I hit things in, four steps.

First, don't take the default behavior of `npm i -g`: either write `@next`, or write the full version number. Second, check first whether the profile has dependencies pointing at `/tmp` or temp directories, and fix them before anything else. Third, under rc.5 those two agent-team packages can be deleted entirely — keeping them only pins the version. Fourth, restart the service after upgrading; don't trust `dsh --version`.

As for whether rc.5's one search change is worth a dedicated upgrade, that depends on whether you use search. Not billing per query saves you in the scenarios where the model triggers queries over and over; if your sessions are mostly reading code and editing files, you won't feel it.

But the batch in rc.2 — Git Worktree review by branch, named multi-level SubAgent — are real features. Anyone parked on rc.1 is short by more than one search line.

Which version is your dsh on right now?
