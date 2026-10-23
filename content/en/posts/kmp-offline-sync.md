---
title: "Row-Level localState Cannot Carry Nested Edits: Reworking Offline Sync with KMP + SQLDelight + Supabase"
date: 2026-09-28T10:11:44+08:00
draft: false
categories: ["WeChat"]
tags: ["Kotlin"]
summary: "A user edits a checklist on the subway: adds two levels, renames one of them, deletes the one they do not want. The whole thing happens offline and only uploads once they are back above ground."
cover:
    image: "/images/kmp-offline-sync/cover.png"
    alt: "Row-Level localState Cannot Carry Nested Edits: Reworking Offline Sync with KMP + SQLDelight + Supabase"
    relative: false
---

## One save that creates, updates and deletes at the same time

A user edits a checklist on the subway: adds two levels, renames one of them, deletes the one they do not want. The whole thing happens offline and only uploads once they are back above ground.

You want that to land on the server as one transaction. But if the client tracks sync state per row, that single save gets split into several independent requests, and if any one of them fails you are left with half-written data in the account.

premi jit chowdhury, the author of Nitivakya, wrote this rework up recently. The app uses KMP + SQLDelight + Supabase and started out as a daily ancient quote, living happily on a localState field (DIRTY / SYNCED / DELETED) on a single favoriteEntity. Then he added Actions, and the structure became a nested tree: Action to Action Levels to Action Level Items.

> A row-level flag tracks final row state, not multi-entity mutation intent.

His words: a row-level flag records the final state of that row, not the intent of a multi-entity change. That sentence is where the rework starts.

## Three specific collapses

He lists three failures in the post. I will quote them first, then unpack.

> 2 separate table REST calls caused foreign-key 500 errors over shaky connections.
> Full-snapshot uploads risked overwriting server-calculated streak progress.
> Row-level flags couldn't express a simultaneous "Create Level A + Update Level B + Delete Level C" edit.

Child rows arrive before parent rows, the server returns a foreign key constraint error, and the child rows stay stuck in DIRTY forever. Route around the split requests with a whole-tree snapshot upload and you overwrite the streak count the server computes itself. The third is the hardest to dodge: creates, updates and deletes mixed into one save gesture cannot be expressed by a row flag at all.

The judgment I agree with is this — the problem is not that the flag holds the wrong value, it is that row granularity cannot carry what actually happened in this edit.

## His new architecture: local state for the UI, an outbox for uploads

The result separates two layers. Local entity tables keep localState, now used only by Compose to render a status dot. Sync moves to a single aggregate outbox table holding sparse JSON DTOs with only the fields that actually changed.

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

That unique constraint is the point: multiple edits to the same entity collapse into one row. On the Kotlin side, withSync does three things inside one database transaction: performs the local write, deep-merges the new payload into the existing payload in the queue, and removes the queue row outright when it sees a create followed by a delete.

Enqueueing uses ON CONFLICT with a state machine guard so an offline create is not mislabeled as an update:

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

Uploads no longer call table by table. One Supabase RPC goes into PostgreSQL. The function locks the row with FOR UPDATE, then runs deletes and upserts in order, and the function body is itself the transaction.

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
    -- then the atomic upsert of levels and JSONB items
END;
$$;
```

## I ran it locally

I ran the article's SQL on sqlite3 3.54.0 (scripts /tmp/kmp-article/outbox.sql and atomicity.sql). All three conclusions hold.

INSERT while offline, then UPDATE the same entity: operation really does stay INSERT, createdAt is not refreshed, and only payload is replaced.

But INSERT followed by a straight DELETE is beyond what the SQL layer handles: the row stays, operation flips to DELETE, and payload is wiped to NULL. The author says that self-cancellation happens in the Kotlin transaction, and the test backs that division of labor up.

The foreign key case reproduced too. Insert into the child table first and you get FOREIGN KEY constraint failed; wrapped in a transaction it rolls back cleanly and neither the parent nor the child table keeps half a row. That is the core reason he insists on RPC rather than per-table REST.

On the Kotlin side I compiled and ran deepMerge and the cancellation logic with the kotlinc from the Gradle cache (kotlin-compiler-embeddable 2.4.10). Four offline edits — create level-3, rename level-3, create level-4, delete level-4 — collapse into one row with one payload:

```
queue row entityId=action-101 operation=INSERT
merged payload={"levelsUpsert":{"level-3":{"title":"Deep Meditation","sequence":3},
"level-4":{"title":"Breath Work","sequence":4}},"levelsDelete":{"level-4":{"id":"level-4"}}}
enqueue count=4, final queue rows=1
```

Create action-202 and then delete it instead, and nothing is left in the queue. Zero network calls.

## One edge I ran into

In that payload above, level-4 appears in both levelsUpsert and levelsDelete. It is created and deleted at the same time.

His deepMerge only merges at key level and does not cancel the same entity across those two maps. So automatic create-then-delete cancellation only covers the INSERT plus DELETE case, the whole-row cancel. A create-then-delete inside the same level has to be handled in order by the server RPC, or pruned by the client during merge.

In other words, the sparse payload buys you out of whole-tree uploads, but it splits conflict resolution across two places: the client transaction and the execution order of the server function. Get either one wrong and you end up with something deleted locally that still exists on the server.

## Four questions to check your own setup

First, does your row-level flag record final state or change intent? If it is the former, a nested structure exposes it immediately.

Second, what happens when a child entity reaches the server before its parent? If the answer is an error and a row stuck in DIRTY, your sync path is missing an atomic entry point.

Third, does a whole-tree snapshot overwrite fields the server computes? Streaks, points and counters are the ones that get silently rewritten.

Fourth, insert-then-delete self-cancellation: do you do it on the client or leave it to the server? Doing it in both places is not bad. Doing it in neither is a data hole.

## Finally

What is worth reading here is not the outbox pattern itself but how clearly the author draws the line on when to change architecture: for simple one-to-one flags, a row-level flag is enough; once nested composite edits appear, store the change intent as a first-class thing.

How do you handle composite offline edits? Tell me your approach in the comments.

References:

- Original post: https://premjitchowdhury.medium.com/when-offline-sync-relying-only-on-local-state-breaks-and-how-i-fixed-it-in-nitivakya-kmp-245e05892ac3
- Reddit thread: https://www.reddit.com/r/Kotlin/comments/1wrdk0q/when_offline_sync_relying_only_on_local_state/
- Supabase database functions and RPC: https://supabase.com/docs/guides/database/functions

Verification boundary: the three SQLite conclusions and the Kotlin merge logic in this article were actually run locally (sqlite3 3.54.0, kotlinc 2.4.10). The Supabase RPC function and PostgREST's real behavior were not verified against a project; the function body is excerpted from the original post.
