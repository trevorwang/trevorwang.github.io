---
layout: post
title: "行级 localState 撑不住嵌套编辑：KMP + SQLDelight + Supabase 离线同步返工记"
date: 2026-09-28 10:11:44 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/kmp-offline-sync/cover.png
---

![行级 localState 撑不住嵌套编辑：KMP + SQLDelight + Supabase 离线同步返工记](/assets/images/kmp-offline-sync/cover.png)

## 一次保存里，同时新建、修改和删除

用户在地铁里改一份清单：新建两层、给其中一层改名、把不要的那一层删掉。整个过程离线，回到地面才上传。

你希望数据到服务端时是一笔事务。可如果客户端是按行标记同步状态的，这一笔就会被拆成几笔各自为政的请求，其中任何一笔失败，账号里就留下半截数据。

Nitivakya 的作者 premi jit chowdhury 最近把这次返工写成了文章。这个 App 用 KMP + SQLDelight + Supabase，原本只是每日一句古语，靠一张 favoriteEntity 上的 localState 字段（DIRTY / SYNCED / DELETED）就活得挺好。直到他加了 Actions，结构变成 Action 到 Action Levels 再到 Action Level Items 的嵌套树。

> A row-level flag tracks final row state, not multi-entity mutation intent.

他的原话是，行级标记记录的是这一行的最终状态，不是多实体变更的意图。这句话是整个返工的起点。

## 三处具体的塌方

他在帖子里列了三条失败，我先照抄再消化。

> 2 separate table REST calls caused foreign-key 500 errors over shaky connections.
> Full-snapshot uploads risked overwriting server-calculated streak progress.
> Row-level flags couldn't express a simultaneous "Create Level A + Update Level B + Delete Level C" edit.

子表先到、父表后到，服务端直接回外键约束错误，子行永久卡在 DIRTY。绕开分散请求、改成整棵快照上传，又会把服务端自己算的连续打卡记录覆盖掉。第三条最难绕：一个保存手势里的增删改混在一起，行标记根本表达不了。

我认同的判断是，问题不在于状态标错了，而在于行级标记这个粒度天然承载不了「这次编辑到底发生了什么」。

## 他的新架构：本地态给 UI，outbox 管上传

改完之后的形态是两层分开。本地实体表保留 localState，只用来给 Compose 渲染状态点；同步交给一张汇总的 outbox 表，里面存稀疏 JSON DTO，只带这次真正改动的字段。

```sql
CREATE TABLE syncQueueEntity (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  userId TEXT NOT NULL,
  type TEXT NOT NULL,
  operation TEXT NOT NULL,
  entityId TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'PENDING',
  createdAt INTEGER NOT NULL,
  payload TEXT,
  UNIQUE(userId, type, entityId)
);
```

那个唯一约束是关键，同一实体的多次编辑被收敛到一行。Kotlin 侧的 withSync 在同一个数据库事务里做三件事：执行本地写入、把新 payload 深合并进队列里的旧 payload、碰上先建后删就把队列行直接删掉。

入队用 ON CONFLICT 加一层状态机守卫，避免把「离线新建」误标成「更新」：

```sql
INSERT INTO syncQueueEntity (userId, type, operation, entityId, createdAt, payload)
VALUES (?, ?, ?, ?, ?, ?)
ON CONFLICT(userId, type, entityId) DO UPDATE SET
  operation = CASE
      WHEN syncQueueEntity.operation = 'INSERT' AND excluded.operation = 'UPDATE' THEN 'INSERT'
      ELSE excluded.operation
  END,
  status = 'PENDING',
  createdAt = syncQueueEntity.createdAt,
  payload = excluded.payload;
```

上传时不再分表调用，而是一个 Supabase RPC 进 PostgreSQL。函数体里先 FOR UPDATE 锁住那一行，再依次做删除与 upsert，整个函数本身就是一个事务。

```sql
CREATE OR REPLACE FUNCTION public.update_action(
    p_action_id uuid, p_updated_at timestamptz,
    p_action_updates jsonb DEFAULT NULL,
    p_levels_upsert jsonb DEFAULT NULL,
    p_levels_delete uuid[] DEFAULT NULL,
    p_items_upsert jsonb DEFAULT NULL,
    p_items_delete uuid[] DEFAULT NULL)
RETURNS void LANGUAGE plpgsql SECURITY INVOKER AS $$
BEGIN
    PERFORM 1 FROM public.actions WHERE id = p_action_id AND user_id = auth.uid() FOR UPDATE;
    IF p_levels_delete IS NOT NULL THEN
        DELETE FROM public.action_levels WHERE id = ANY(p_levels_delete);
    END IF;
    -- 接着是 levels 与 JSONB items 的原子 upsert
END;
$$;
```

## 我拿本机跑了一遍

文章里的 SQL 我照着在 sqlite3 3.54.0 上跑了一遍（脚本 /tmp/kmp-article/outbox.sql 与 atomicity.sql），三段结论都对得上。

先在离线状态 INSERT，再对同一实体 UPDATE，operation 确实还是 INSERT，createdAt 也没被刷新，只有 payload 换成了新的。

而 INSERT 之后直接 DELETE，SQL 这一层兜不住：行还在，operation 变成 DELETE，payload 被 NULL 冲掉。作者说这种自我抵消是在 Kotlin 事务里做的，实测坐实了这个分工。

外键那条也复现了。先插子表得到 FOREIGN KEY constraint failed，放进事务里则回滚得干干净净，父表子表都没留下半行。这就是他坚持走 RPC 而不是分表 REST 的核心理由。

Kotlin 侧我用 Gradle 缓存里的 kotlinc（kotlin-compiler-embeddable 2.4.10）编译跑了 deepMerge 与抵消逻辑。四次离线编辑，新建 level-3、给 level-3 改名、新建 level-4、删掉 level-4，最后收敛成一行一条 payload：

```
队列行 entityId=action-101 operation=INSERT
合并后 payload={"levelsUpsert":{"level-3":{"title":"Deep Meditation","sequence":3},
"level-4":{"title":"Breath Work","sequence":4}},"levelsDelete":{"level-4":{"id":"level-4"}}}
入队次数=4，最终队列行数=1
```

换成新建 action-202 再删除，队列里一行都不剩，网络调用是零。

## 我实测撞上的一个边界

上面那条 payload 里，level-4 同时出现在 levelsUpsert 和 levelsDelete 里，它既被创建又被删除。

作者的 deepMerge 只做键级合并，不会跨这两个 map 去抵消同一个实体。所以「先建后删」的自动抵消只覆盖了 INSERT 加 DELETE 这一层，也就是整行取消；同一层里「建了又删」要靠服务端 RPC 按顺序处理，或者客户端在合并时自己剔除。

换句话说，稀疏 payload 省掉了整树上传的代价，但它把一部分冲突消解的活分给了两个地方：客户端的事务，和服务端函数的执行顺序。这两处只要有一处逻辑不齐，就会出现本地看着删了、服务端还留着的状态。

## 拿去自查的四条

第一，你的行级标记记录的是最终状态还是变更意图。如果是前者，嵌套结构一上来就会露馅。

第二，子实体先于父实体到达服务端会怎样。如果答案是报错然后卡在 DIRTY，那你的同步链路上少了一个原子入口。

第三，整棵快照上传会不会覆盖服务端算出来的字段。打卡、积分、统计这类字段最容易被静默改写。

第四，插入又删除这种自我抵消，你是在客户端做的还是留给服务端。两边都做不算坏事，两边都没做就是数据缺口。

## 最后

这次返工值得看的地方，不是 outbox 这个模式本身，而是作者把「什么时候该换架构」写清楚了：简单的一对一标记，行级 flag 够用；一旦出现嵌套的复合编辑，就该把变更意图当成一等公民存起来。

你怎么处理复合离线编辑？欢迎在评论区聊聊你的方案。

参考资料：

- 原文：https://premjitchowdhury.medium.com/when-offline-sync-relying-only-on-local-state-breaks-and-how-i-fixed-it-in-nitivakya-kmp-245e05892ac3
- 原帖：https://www.reddit.com/r/Kotlin/comments/1wrdk0q/when_offline_sync_relying_only_on_local_state/
- Supabase 数据库函数与 RPC：https://supabase.com/docs/guides/database/functions

验证边界：文中 SQLite 的三段结论与 Kotlin 的合并逻辑在本机真跑过（sqlite3 3.54.0、kotlinc 2.4.10）；Supabase RPC 函数与 PostgREST 的真实行为没有连项目验证，函数体摘录自原文。
