---
title: "CLAUDE.md 之后是什么：新项目先别写"
date: 2026-10-12T08:03:05+08:00
draft: false
categories: ["公众号"]
tags: ["AI Agent"]
summary: "一个多月前，Claude Code 因为拒绝兼容 AGENTS.md（已经被很多项目和 coding agent 采用的行业约定），被 Shopify 的 CEO Tobi Lütke 公开警告要弃用它。Anthropic 当时的理由听着也合理：不同模型需要不同的上下文和指令。"
cover:
    image: "/images/claude-md/cover.png"
    alt: "CLAUDE.md 之后是什么：新项目先别写"
    relative: false
---

## 先说一件反常的事

一个多月前，Claude Code 因为拒绝兼容 AGENTS.md（已经被很多项目和 coding agent 采用的行业约定），被 Shopify 的 CEO Tobi Lütke 公开警告要弃用它。Anthropic 当时的理由听着也合理：不同模型需要不同的上下文和指令。

一个多月后，Claude Code 团队确认要支持 AGENTS.md 了。但团队里负责这块的 Thariq Shihipar 在 Latent Space 播客里给出的下一步更激进：

> 我认为，在极限情况下，CLAUDE.md 是会消失的。甚至可能不用等到那么远，我觉得现在开始一个新项目，可能最好就是不要 CLAUDE.md。

## 他的四条理由

第一条，模型的下限在涨：

> 随着模型越来越好，它们完成最简单任务的下限也在不断提高。

第二条是最实用的一条，他讲的是顺序：先让它跑，只有反复看到同一个失败模式，再把规则加进去。

第三条反直觉：

> 如果你一直保留着这份不断增长的失败模式日志，它们很可能会过度约束 Claude。

第四条讲维护成本。这件事因模型而异，你可能需要 FABLE.md，需要 OPUS.md，甚至同一模型的两个版本之间都不一样。他的原话是「维护不同的文件实在是太痛苦了」。

## 那之后该写什么

访谈里主持人 swyx 点出两样东西，说它们重要到不该只放在提示词里：一个是 Goals，项目要往哪去；另一个是决策日志、实验日志这类要活过单次会话的外部产物。Thariq 在回答里承认 AGENTS.md 他们还是要做的，但同一段话里，他把维护多份文件称为最痛苦的部分。

顺着这个逻辑，约定文件的角色会从禁止清单转向目标与记录：告诉模型这条路通向哪里，把做过的判断留在文件里，而不是把二十条不要做什么堆在开头。禁止清单是给旧模型的边界打的补丁，目标不是。

## 本机实测：约定文件到底是怎么被读进去的

我天天在几个不同的 agent 客户端之间切，所以把本机的加载规则查了一遍，有四件事值得写出来。

第一，AGENTS.md 这条支持是刚补上的。我翻了官方更新日志，它出现在 2.1.277：

> Added AGENTS.md support: in a project with no CLAUDE.md, Claude Code reads AGENTS.md instead; change it under "Project instructions" in /config

注意它的定位：项目里没有 CLAUDE.md 的时候，才读 AGENTS.md。它是后备，不是并列。而我本机装的是 2.1.150，比这个版本旧，所以这台机器上的 Claude Code 至今不认 AGENTS.md。

第二，另一个客户端（我日常用的是 Hermes，就是现在在飞书里跟我说话的这个）路子完全不同。它按优先级扫当前目录下的候选文件，同一层只让一个赢，其余的标成 shadowed：

> not loaded — higher-priority context type wins

第三，它明确拒绝读别的工具目录下的文件。源码里写得很直白：子目录提示会拒绝工作目录之外的路径，所以 ~/.codex/AGENTS.md 和 ~/.claude/CLAUDE.md 永远不会混进来。我本机那份 ~/.claude/CLAUDE.md 里只有 226 个字符（一条 /graphify 的路由规则），Hermes 从来没读过它。

第四，加载有上限。子目录提示最多带 32,000 个字符，超了头尾截断。而我那套项目里的根文件是 36,631 字符，后半截一直在被裁掉。

## 量一下这笔账

我用本机 agent 自带的分词估算跑了一遍：

```
   7151 字符  ~  1794 tokens  ~/AGENTS.md
    226 字符  ~    57 tokens  ~/.claude/CLAUDE.md
   9387 字符  ~  2357 tokens  ~/kmp-shop/CLAUDE.md
  36631 字符  ~  9234 tokens  ~/.hermes/hermes-agent/AGENTS.md

hermes-agent 仓库里的 AGENTS.md：12 个文件
合计 169,982 字符（约 42,496 tokens）
```

一个仓库，12 份约定文件，加起来四万多 tokens。它写得其实挺克制，根文件自己就写着「只放全局通用的，每个目录有自己的 AGENTS.md，目标 8k 字符」，但数字摆在这，每次开工都要背一遍。

![](/images/claude-md/fig-read.png)

## 我的判断

Thariq 说的是别囤，不是别写。这两件事经常被转述成一件，然后大家要么把文件删光，要么继续往里加。

我给自己的判断法就一条：这个规则如果对应一次具体的失败，留下；如果只对应「我担心它会做错」，那是猜，先别写。前者是模型真犯过的错，后者是我在替模型想象边界。

还有一层是分层。别把全库的规矩塞进一个根文件，那正是我上面那 36,631 字符的来历。真正生效的做法反而是每个目录放一份小的，谁在哪个目录干活就加载哪一份。

顺带说个现成的例子：这份根文件里写着「目标 8k 字符」，它自己长到了 36k。约定文件是需要有人定期删东西的，不然它会自己长胖。

你项目里那份约定文件，最近一次删内容是什么时候？欢迎评论区聊聊。
