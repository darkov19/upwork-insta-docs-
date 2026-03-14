# Algorithm Decisions

## Instagram Monitoring SaaS

---

## 1. Follower/Following Diff Algorithm

### Problem

After each sync, we need to know exactly which accounts followed and which unfollowed the monitored target since the last sync.

### Approach: State Table Diff (Set Subtraction)

We maintain a persistent state table (`followers_state`, `following_state`) representing the **last known set** of follower/following account IDs per target. On each sync we fetch the current set from HikerAPI and compute the diff in-database.

**Why not store full snapshots?**

- Snapshots accumulate fast (account with 10k followers × 10 syncs = 100k rows of redundant data)
- State tables hold only the current set — O(n) space not O(n×syncs)
- Diff is simple set subtraction, computed once per sync

### Algorithm

```
prev_set    = SELECT ig_user_id FROM followers_state WHERE target_id = ?
current_set = HikerAPI paginated fetch → Set of ig_user_ids

added   = current_set - prev_set   → FOLLOWER_ADDED events
removed = prev_set - current_set   → FOLLOWER_REMOVED events

-- Write events
INSERT INTO events (event_type = FOLLOWER_ADDED)   FOR each in added
INSERT INTO events (event_type = FOLLOWER_REMOVED) FOR each in removed

-- Diff-only state update (do NOT delete all + insert all)
INSERT INTO followers_state (target_id, ig_user_id, ig_username)
  VALUES ... FOR each in added
  ON CONFLICT DO NOTHING

DELETE FROM followers_state
  WHERE target_id = ? AND ig_user_id = ANY(removed_ids_array)
```

Same logic applies for `following_state` / `FOLLOWING_*` events.

### Why Diff-Only Writes — Not DELETE All + INSERT All

The naive approach is to delete all rows for the target and re-insert the full current set. This is simple but catastrophically expensive at scale:

```
DELETE all + INSERT all:
  20k follower account = 20k deletes + 20k inserts = 40,000 row ops per sync
  600 syncs/day at 1000 users = 24 million row ops/day on followers_state alone

Diff-only:
  Typical sync: ~100 follows + ~50 unfollows = 150 row ops per sync
  600 syncs/day = 90,000 row ops/day — 266× reduction
```

The diff is computed in memory inside the Edge Function (current_set and prev_set are both loaded as sets), then only the delta is written to the database.

### Complexity

- **Time**: O(n + m) where n = previous set size, m = current set size
- **Space**: O(n) state per target (only current snapshot)
- **DB operations**: 1 read, 1 conditional bulk insert (added only), 1 conditional bulk delete (removed only)

### Idempotency

The sync is idempotent: if the same sync runs twice (e.g., retry after crash), the second run produces the same diff because we replace the state atomically. We use `sync_run_id` to detect and skip already-completed runs.

```sql
-- Guard: skip if sync_run already completed
IF EXISTS (SELECT 1 FROM sync_runs WHERE id = ? AND status = 'completed') THEN
  RETURN;
END IF;
```

---

## 2. Sync Rate Limiting

### Two Independent Limits

#### a. Per-Target Cooldown

Prevents syncing the same target too frequently (protects HikerAPI costs).

```
last_synced_at = SELECT last_synced_at FROM targets WHERE id = ?

IF now() - last_synced_at < cooldown_minutes THEN
  RETURN 429 "Cooldown active, retry after X minutes"
END IF
```

Cooldown value comes from the user's active plan (`plans.cooldown_minutes`).

#### b. Per-User Daily Sync Limit

Caps total syncs across all targets per user per calendar day.

```
syncs_today = SELECT COUNT(*) FROM sync_runs
  WHERE user_id = ?
    AND status IN ('completed', 'running')
    AND started_at >= today_start_utc

IF syncs_today >= plan.daily_sync_limit THEN
  RETURN 429 "Daily sync limit reached"
END IF
```

All checks happen **before** the sync starts, in order:

1. Target cooldown check
2. Daily limit check
3. Concurrent sync lock (see below)
4. Proceed

#### c. Concurrent Sync Guard — Race Condition Fix

A naive status check (`WHERE status = 'active'`) has a race condition: two requests can both read `status = 'active'` before either writes `status = 'syncing'`. Both proceed, running duplicate syncs and producing duplicate events.

Fix: use PostgreSQL's `FOR UPDATE SKIP LOCKED` to atomically acquire the target row as a lock:

```sql
-- Returns the row only if it is not already locked by another transaction
SELECT id FROM targets
WHERE id = ? AND status = 'active'
FOR UPDATE SKIP LOCKED
```

If the row is already locked by another sync, this query returns zero rows — the second request gets a clean 409 Conflict without ever touching the sync logic. The lock is held for the duration of the transaction that sets `status = 'syncing'`, then released.

---

## 3. HikerAPI Page Cap

### Problem

Accounts with large follower counts consume many API request units. Without a cap, a single sync on a 500k-follower account could exhaust the user's allowance or cause runaway cost.

### Decision: Hard Page Cap Per Sync

```
MAX_PAGES_PER_SYNC = plan.page_cap  (configurable per plan tier)

pages_fetched = 0
cursor = null

LOOP:
  response = HikerAPI.getFollowers(ig_user_id, cursor)
  pages_fetched += 1
  accumulate results

  IF pages_fetched >= MAX_PAGES_PER_SYNC THEN
    BREAK  -- truncate, mark sync as partial
  END IF

  IF response.next_cursor == null THEN
    BREAK  -- natural end
  END IF

  cursor = response.next_cursor
```

If the sync is truncated (hit page cap before completion), `sync_runs.status = 'completed_partial'`. The state table still reflects what was fetched — a partial sync is better than a failed one.

### Plan Tier → Page Cap → Job Duration

The plan's `max_follower_count` and `page_cap` work together to bound Edge Function execution time within the 150-second limit:

| Plan       | Max Followers | Page Cap  | Est. Followers Fetched | Est. Duration | Within 150s? |
| ---------- | ------------- | --------- | ---------------------- | ------------- | ------------ |
| Basic      | 50k           | 25 pages  | 5,000                  | ~15s          | ✅           |
| Pro        | 500k          | 100 pages | 20,000                 | ~70s          | ✅           |
| Enterprise | 1M            | 200 pages | 40,000                 | ~140s         | ✅ barely    |

MVP cap: **1 million followers maximum** across all plans. Accounts exceeding this are rejected at add time. Beyond 1M, the async architecture needs to be replaced with a managed job queue (Trigger.dev / Inngest) — a post-MVP upgrade.

### Why not reject large accounts entirely?

We already reject accounts exceeding `plan.max_follower_count` at **add time**. The page cap is a secondary defense for accounts that grew after being added, or for plans that allow larger accounts with higher page caps.

---

## 4. Async Sync Execution

### Problem

Syncing a large account (e.g. 500k followers, 100 pages) takes 60–120 seconds. An HTTP request cannot stay open that long — Vercel serverless functions timeout at 10–60 seconds, and the user's browser would show a hanging request.

### Decision: Inngest + Supabase Realtime

The `/api/sync` route does the minimum — validates, creates the job record, sends an Inngest event, and returns 202 immediately. Inngest executes the sync function in the background with **no timeout limit**. The frontend subscribes to the job record via Supabase Realtime and receives a push update when the job completes.

```
POST /api/sync
    ├── validate (auth, cooldown, limits, FOR UPDATE SKIP LOCKED)
    ├── create sync_run { status: 'queued' }
    ├── inngest.send('sync/run', { syncRunId, targetId, planConfig })
    └── return 202 { syncRunId }     ← closes in < 500ms

Frontend:
    supabase
      .from('sync_runs')
      .on('UPDATE', { filter: `id=eq.${syncRunId}` }, payload => {
          if (payload.new.status === 'completed') showResults()
          if (payload.new.status === 'failed')    showError()
      })
      .subscribe()

Inngest function executes (no timeout — runs until done):
    └── on completion → UPDATE sync_run.status = 'completed'
                     → Realtime fires → frontend updates instantly
```

### Why Inngest over Supabase Edge Functions?

|                                | Supabase Edge Functions  | Inngest                      |
| ------------------------------ | ------------------------ | ---------------------------- |
| Timeout                        | 150 seconds              | No limit                     |
| Rate limiting                  | Manual DB semaphore hack | Built-in, declared in config |
| Retries on failure             | Manual try/catch         | Built-in, configurable       |
| Concurrency control            | None                     | Built-in, per-function limit |
| Monitoring                     | Query sync_runs table    | Full dashboard with logs     |
| Auto sync support              | Breaks at scale          | Native fan-out + queue       |
| Cost (manual sync, 1000 users) | Supabase Pro required    | Free tier covers it          |

### Why not polling?

Polling (`GET /api/sync/status` every 3 seconds) generates unnecessary requests. Supabase Realtime is a persistent WebSocket — zero extra requests, instant update the moment Inngest writes the completion status.

---

## 5. HikerAPI Rate Limiting via Inngest

### Problem

At 10–15 concurrent sync jobs (or 50+ with auto sync), all sharing one HikerAPI API key, the combined request rate can exceed HikerAPI's per-key rate limit.

### Decision: Inngest Built-in Rate Limiting

Rate limiting is declared directly in the Inngest function configuration — no DB row, no manual semaphore, no custom retry logic:

```typescript
inngest.createFunction(
    {
        id: "sync-target",
        rateLimit: {
            limit: 20, // max 20 HikerAPI requests per second
            period: "1s", // across ALL concurrent sync jobs globally
        },
        retries: {
            attempts: 3,
            backoff: "exponential", // 1s → 2s → 4s automatically on 429
        },
        concurrency: {
            limit: 50, // max 50 sync functions running simultaneously
        },
    },
    { event: "sync/run" },
    async ({ event, step }) => {
        /* sync logic */
    },
);
```

Inngest enforces these limits globally across all running function instances. If 50 jobs are running and they collectively hit 20 req/s, Inngest queues the excess requests automatically — no application code needed.

### Jitter

Each page fetch still adds a small random delay (100–300ms) to distribute requests within the rate window and avoid burst collisions at the start of each second window:

```typescript
await step.sleep("jitter", Math.random() * 300);
const page = await hikerApi.getFollowers(cursor);
```

---

## 7. Target Status State Machine

```
                  ┌─────────┐
                  │  ADDING │ (validation in progress)
                  └────┬────┘
                       │ pass validation
                       ▼
                  ┌────────┐
         ┌───────▶│ ACTIVE │◀────────────────┐
         │        └───┬────┘                 │
         │            │                      │
         │    ┌───────┼──────────┐           │
         │    ▼       ▼          ▼           │
         │ SYNCING  PAUSED    PAUSED       PAUSED
         │         PRIVATE   OUT_OF_PLAN   ERROR
         │    │
         │    ├── sync complete → ACTIVE
         │    └── sync failed   → PAUSED_ERROR
         │
         └──── user retries or issue resolves (manual re-activate)
```

### Status Transitions

| Trigger                               | New Status           |
| ------------------------------------- | -------------------- |
| Sync starts                           | `syncing`            |
| Sync completes successfully           | `active`             |
| HikerAPI returns account is private   | `paused_private`     |
| Follower count now exceeds plan limit | `paused_out_of_plan` |
| HikerAPI error (non-transient)        | `paused_error`       |
| User removes target                   | deleted              |

**Private/out-of-plan detection happens during sync**, not in a background job. The next manual sync is when we discover the account changed.

---

## 8. Event Retention (90-Day Cleanup)

Detailed events are kept for 90 days. A pg_cron job runs daily to purge old records.

```sql
-- Runs daily at 02:00 UTC
SELECT cron.schedule('purge-old-events', '0 2 * * *', $$
  DELETE FROM events
  WHERE detected_at < now() - INTERVAL '90 days';
$$);
```

`daily_stats` is kept permanently — it's aggregated and cheap (one row per target per day).

---

## 9. Cost Safeguard: Request Unit Budget per Sync

### Problem

HikerAPI charges per request unit. We need to prevent a single sync from exceeding a cost threshold.

### Decision: Unit Budget Check Mid-Sync

```
units_used = 0
UNIT_BUDGET = plan.max_units_per_sync  (configurable)

FOR each page fetch:
  response = HikerAPI.getFollowers(...)
  units_used += response.request_units_consumed

  IF units_used > UNIT_BUDGET THEN
    mark sync as partial
    BREAK
  END IF
```

After each completed sync:

```sql
INSERT INTO request_unit_usage (user_id, sync_run_id, units)
VALUES (?, ?, ?)
```

This gives us per-user cost visibility and lets us detect users whose usage patterns are unprofitable.

---

## 10. Story TTL Caching

### Problem

Multiple users may monitor the same Instagram account and view its stories. Without caching, each user triggers a separate HikerAPI fetch — wasting request units on identical content.

### Decision: Dynamic TTL Based on Story Age

Instagram stories expire exactly 24 hours after their `taken_at` timestamp. HikerAPI returns `taken_at` (Unix timestamp) for each story item. We use this to compute a precise remaining TTL and emit it as a `Cache-Control` header. Vercel's edge cache handles the rest.

```
taken_at     = story.taken_at          (Unix seconds from HikerAPI)
expires_at   = taken_at + 86400        (24h in seconds)
remaining_s  = expires_at - now()      (seconds until story expires on Instagram)
remaining_s  = max(remaining_s, 60)    (floor at 60s to avoid zero/negative)

Cache-Control: public, s-maxage=<remaining_s>, max-age=<remaining_s>
```

### How it works end-to-end

```
User A → GET /api/stories/[targetId]
  → auth check passes
  → plan check passes
  → HikerAPI fetch: stories for ig_user_id
  → compute remaining_ttl per story
  → set Cache-Control: s-maxage=<remaining_ttl>
  → Vercel edge caches the response
  → return stories to User A

User B → GET /api/stories/[targetId]  (same target, 2 minutes later)
  → Vercel edge cache HIT
  → response served without touching HikerAPI
  → 0 request units consumed
```

### Why this is better than a fixed cache duration

- A story uploaded 23h 50m ago should only be cached for 10 minutes — a fixed 10-minute cache would work but wastefully re-fetch content that will expire anyway
- A story uploaded 1 hour ago gets cached for 23 hours — maximum efficiency
- No manual invalidation logic needed — the math is self-correcting
- No extra infrastructure — Vercel edge cache is free and already in the stack

### Cache key

The cache key is the full request URL: `/api/stories/[targetId]`. If the target has a new set of stories on the next request (after cache expires), a fresh fetch automatically occurs.

### No permanent storage

Media content is never written to disk or database. The edge cache is ephemeral — content evicts automatically when TTL expires or under memory pressure.

---

## 11. Username Tracking

Instagram `pk` (user ID) is stable; `username` can change. We store both.

```
On each sync:
  current_username = HikerAPI response for ig_user_id

  IF current_username != targets.ig_username THEN
    UPDATE targets SET ig_username = current_username
  END IF
```

Events always store both `ig_user_id` and `ig_username` at detection time. If a username changes after the event is recorded, we have the historical username in the event row.

---

## 12. Weekly Email Decision Logic

```
FOR each user:
  last_login = SELECT MAX(last_sign_in_at) FROM auth.users WHERE id = user.id

  IF last_login < 7 days ago THEN
    SKIP  -- user is inactive
  END IF

  already_sent = SELECT 1 FROM email_log
    WHERE user_id = user.id AND week_start = this_monday

  IF already_sent THEN
    SKIP  -- idempotent
  END IF

  activity = SELECT SUM(followers_gained + followers_lost + ...)
    FROM daily_stats
    WHERE target_id IN (user's targets)
      AND date >= this_monday - 7 days

  IF activity == 0 THEN
    SKIP  -- nothing to report
  END IF

  SEND email with per-target breakdown
  INSERT INTO email_log (user_id, week_start)
```

The `email_log` insert at the end acts as a distributed lock — if the cron fires twice, the second run finds the row and skips. This makes the job idempotent.
