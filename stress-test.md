# Stress Test Analysis

## Instagram Monitoring SaaS — 1000 Concurrent Enterprise Users

---

## Scenario

> **1,000 users, all on the highest plan (Enterprise), all clicking sync at the exact same moment.**

This is the theoretical worst case — manual sync requires users to actively click a button, so this scenario does not occur naturally. It becomes realistic only with auto sync enabled (cron fires → all targets queued simultaneously), which is why auto sync is deferred from MVP.

**Enterprise plan assumptions:**

- Max followers per target: 1,000,000
- Page cap: 200 pages per sync
- Followers fetched per sync: ~40,000
- Job duration (solo, no contention): ~140s

---

## Layer-by-Layer Analysis

### Layer 1 — Vercel / Next.js (`POST /api/sync`)

1,000 requests hit `/api/sync` simultaneously. Vercel spins up 1,000 serverless function instances in parallel. Each does:

```
Auth check              → Supabase query
Cooldown check          → Supabase query
Daily limit check       → Supabase query
FOR UPDATE SKIP LOCKED  → Supabase query (atomic row lock)
INSERT sync_run         → Supabase write
inngest.send()          → Inngest HTTP call
Return 202              → closes in < 500ms
```

**Verdict: Holds.** Vercel is designed for horizontal scale — 1,000 parallel serverless invocations is within its design envelope.

---

### Layer 2 — Supabase Database

1,000 functions each need DB connections simultaneously. Supabase Pro uses PgBouncer in **transaction mode** — connections are multiplexed, not dedicated. Each query borrows a connection for milliseconds then releases it.

```
1,000 functions × 5 queries = 5,000 DB queries in a tight burst
PgBouncer transaction mode: ~25 server connections, serving many clients
```

**Key protection — `FOR UPDATE SKIP LOCKED`:** Each user's target is a different row, so there is no cross-user lock contention. All 1,000 requests lock different rows simultaneously — no queue forms at the application layer.

**Realistic outcome:** PgBouncer handles the burst in transaction mode. Under true simultaneous load, 5–10% of `/api/sync` calls may return 500 if the connection queue backs up momentarily. Client-side retry resolves these — the request is idempotent up to the `inngest.send()` call.

**Verdict: Holds with minor degradation. ~5% transient error rate at peak burst, resolved by retry.**

---

### Layer 3 — Inngest Queue ⚠️ Primary Bottleneck

All 1,000 `sync/run` events land in Inngest. Current function config:

```typescript
concurrency: { limit: 50 }       // 50 jobs run simultaneously
rateLimit:   { limit: 20, period: '1s' }  // 20 HikerAPI req/s globally
```

50 jobs start immediately. 950 are queued correctly — Inngest does not crash or drop jobs.

#### Job duration under full concurrency

The global rate limit is shared across all 50 running jobs:

```
Rate per job = 20 req/s ÷ 50 jobs = 0.4 req/s per job
Enterprise page cap = 200 pages

Time per job = 200 pages ÷ 0.4 req/s = 500 seconds ≈ 8.3 minutes
```

Compare: solo job = 140s. Full concurrency = 500s. **Rate limiting multiplies job duration by 3.5× under full concurrency.**

#### Queue drain time

```
1,000 jobs ÷ 50 concurrent = 20 batches
Each batch ≈ 500s (rate-limited)
Total drain time = 20 × 500s = 10,000s ≈ 2.8 hours
```

A user who clicks sync at peak load waits up to **2.8 hours** for results.

**Verdict: Queue is correct — no crashes, no dropped jobs. User experience is severely degraded. This is the primary architectural bottleneck under this scenario.**

---

### Layer 4 — HikerAPI

```
1,000 users × 200 pages = 200,000 API requests in one mass event
At 20 req/s cap: 200,000 ÷ 20 = 10,000 seconds = 2.8 hours to process
```

**Cost implication:** 200,000 request units consumed in a single sync wave. The `unit_budget_per_sync` guard (Section 9 of algorithm-decisions.md) caps cost per individual sync — this guard is critical and must remain enforced.

**Verdict: Holds — rate limiting caps the request rate correctly. Cost is bounded by per-sync budget guard.**

---

### Layer 5 — Supabase Storage

`followers_state` table size scales with actual follower counts per account:

| Plan       | Avg followers per account | followers_state at 1,000 users |
| ---------- | ------------------------- | ------------------------------ |
| Basic      | 50,000                    | ~2.5 GB                        |
| Pro        | 500,000                   | ~25 GB                         |
| Enterprise | 1,000,000                 | **~50 GB**                     |

The architecture notes estimate ~3.2GB at 1,000 users — that assumed average accounts (~20k followers). Enterprise 1M-follower accounts are **15× larger**.

**Storage cost impact:**

```
Supabase Pro includes: 8 GB
Enterprise scenario:   50 GB
Overage:               42 GB × $0.125/GB = ~$5.25/month extra
```

Cost is manageable. The concern is query latency — reading 1M rows from `followers_state` for a single target takes 3–8 seconds per sync (see Layer 6).

**Verdict: Cost impact is minor (~$5/month). Query latency on very large accounts adds seconds to job runtime but does not break anything.**

---

### Layer 6 — Inngest Function Memory (Corrected Assessment)

#### Where the function actually runs

Inngest is an **orchestrator, not a compute host**. When Inngest triggers a job, it sends an HTTP callback to your `/api/inngest` endpoint on **Vercel**. The function executes inside a Vercel serverless function instance:

```
Inngest (scheduler / queue)
    │
    │  POST /api/inngest  ← HTTP callback
    ▼
Vercel Serverless Function  ← code executes here
                              memory is consumed here
```

Upgrading the Inngest plan changes queue capacity and concurrency limits — **not the memory available to your function.** Memory is a Vercel concern.

#### Peak memory per Enterprise sync function

```
prev_set (1M rows from followers_state → JS Set):     ~80 MB
Supabase query result buffer (before Set is built):    ~80 MB  ← peak, then GC'd
current_set (40k entries from 200 HikerAPI pages):    ~3 MB
HikerAPI response buffers (200 pages × ~10 KB):       ~2 MB
Node.js baseline + Inngest SDK:                       ~50 MB
──────────────────────────────────────────────────────────────
Peak total:                                           ~215 MB
```

**Vercel Hobby / Pro memory limit: 1,024 MB.**

Peak estimated usage is ~215 MB — well within the limit. **There is no OOM risk on a standard Vercel plan.**

#### The real concern: query latency, not memory

Loading 1M rows from Supabase into a JavaScript Set takes time:

```
SELECT ig_user_id FROM followers_state WHERE target_id = ?
  → returns 1M rows
  → Supabase JS client buffers full result
  → 3–8 seconds for this single query
```

The sync still completes. It is slower on very large accounts, but the HikerAPI fetching (hundreds of seconds) dominates the runtime — a few extra seconds on the state read is not a meaningful degradation.

**Verdict: No OOM risk. No Inngest or Vercel plan upgrade required for memory. Slow state-read on 1M accounts is a latency nuisance, not a failure.**

---

### Layer 7 — Supabase Realtime

1,000 users each subscribe to their `sync_run` row via WebSocket:

```javascript
supabase
    .from("sync_runs")
    .on("UPDATE", { filter: `id=eq.${syncRunId}` }, callback)
    .subscribe();
```

Supabase Realtime is designed for thousands of concurrent WebSocket connections. Pro plan handles this without configuration changes.

**Verdict: Holds.**

---

## Full Results Summary

| Layer                        | Holds Under 1,000 Concurrent? | Issue                                   | Severity |
| ---------------------------- | ----------------------------- | --------------------------------------- | -------- |
| Vercel (`/api/sync`)         | Yes                           | None                                    | —        |
| Supabase connections         | Mostly                        | ~5% transient error rate at burst       | Low      |
| `FOR UPDATE SKIP LOCKED`     | Yes                           | No cross-user row contention            | —        |
| Inngest queue                | Yes (queues correctly)        | Up to 2.8-hour wait time                | **High** |
| HikerAPI rate limiting       | Yes                           | Cost if budget guard removed            | Medium   |
| followers_state storage      | Yes                           | ~50 GB at full Enterprise scale         | Low      |
| followers_state read latency | Yes                           | 3–8s extra per sync on 1M accounts      | Low      |
| Inngest function memory      | Yes                           | No OOM — ~215 MB peak vs 1,024 MB limit | None     |
| Supabase Realtime            | Yes                           | Designed for 1,000s of WS connections   | —        |

---

## Why the Worst Case Doesn't Occur in Practice

The 2.8-hour queue drain requires **1,000 users all clicking sync at the exact same second.** Manual sync is user-initiated — in practice, syncs are distributed throughout the day.

**Realistic peak concurrency at 1,000 users (manual sync):**

```
Assumptions: 20% of users sync on any given day
             Syncs distributed across 12 active hours
             200 users/day ÷ 12h ÷ 60min × 5min clustering = ~10 concurrent
```

At 10 concurrent Enterprise syncs:

```
Rate per job = 20 req/s ÷ 10 jobs = 2 req/s
200 pages ÷ 2 req/s = 100 seconds per job
Queue drains in minutes, not hours
```

The architecture handles realistic load comfortably. The theoretical worst case only materialises with **auto sync** (cron fires → all targets queued simultaneously) — which is why auto sync is deferred from MVP.

---

## Required Actions Before Enabling Auto Sync

Auto sync transforms the theoretical worst case into the guaranteed steady state:

| Action                                                       | Why                                     |
| ------------------------------------------------------------ | --------------------------------------- |
| Increase Inngest concurrency limit (50 → higher)             | Reduce queue wait time                  |
| Negotiate higher HikerAPI rate limit (20 req/s → higher)     | Reduce job duration under load          |
| Add UI messaging for queue position / estimated wait         | User expectation management             |
| Price auto sync frequency into subscription tiers            | Inngest cost scales with sync frequency |
| Verify Supabase Pro handles 50 GB at expected query patterns | Storage and index health                |

---

## Required Actions Before Launching Enterprise Tier

Enterprise is safe to launch post-MVP once these are confirmed:

| Action                                                         | Blocks                              |
| -------------------------------------------------------------- | ----------------------------------- |
| Benchmark followers_state read for 1M-row targets              | Confirm 3–8s estimate is acceptable |
| Confirm HikerAPI request unit pricing at 200k units/sync wave  | Cost model for Enterprise pricing   |
| Stream followers_state read in batches if latency unacceptable | Performance optimisation            |

---

## MVP Recommendation

Launch with **Basic and Pro tiers only.** This sidesteps all medium and high severity issues:

- Pro tier: 500k max followers, 100 pages, ~70s solo job duration
- At 10 concurrent Pro syncs: ~35s per job (2 req/s each) — fast, responsive
- followers_state: ~25 GB at full Pro scale — within Supabase Pro storage budget
- No OOM concern, no meaningful queue delay under realistic load

Enterprise tier and auto sync are post-MVP features, added once the infrastructure has been observed under real load.
