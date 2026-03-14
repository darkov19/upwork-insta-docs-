# Architecture

## Instagram Monitoring SaaS

---

## Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend + API | Next.js (App Router) | Single repo for landing page, auth, and dashboard; API routes replace a separate backend |
| Hosting | Vercel | Zero-config Next.js deployment, free tier sufficient; edge cache handles story TTL caching |
| Database | Supabase PostgreSQL | Managed, free tier, Row Level Security built in |
| Auth | Supabase Auth | Handles sessions, tokens, email verification |
| Async sync execution | Inngest | Managed background job platform — no timeout, built-in queuing, concurrency control, retries, monitoring dashboard |
| Real-time sync status | Supabase Realtime | Frontend subscribes to sync_run row — no polling, push update when sync completes |
| Scheduled jobs | Inngest (weekly email) + pg_cron (event cleanup) | Inngest handles sync scheduling and email; pg_cron handles lightweight DB cleanup |
| Instagram data | HikerAPI | Managed scraping layer — no custom Puppeteer/proxy infra |
| Payments | Stripe | Checkout, webhooks, subscription management |
| Email | Resend | Simple API, generous free tier, React email templates |

---

## Domain Structure

```
domain.com          → client's landing page (not our responsibility)
app.domain.com      → our Next.js app (Vercel)
```

Client updates their CTA links to point to `app.domain.com`. That is the only integration required on their side.

## System Diagram

```
domain.com (client owns)
  "Get Started" ──────────────────────────────────┐
  "Login"  ───────────────────────────────────────┤
                                                  │
                       ┌──────────────────────────▼──────────────────┐
                       │     app.domain.com — Vercel (Next.js)       │
                       │                                             │
                       │  /plans             Plan selection          │
                       │  /register          Post-payment signup     │
                       │  /login             Login                   │
                       │  /dashboard         Main app                │
                       │  /dashboard/stories Story viewer            │
                       │                                             │
                       │  /api/sync          Validate + enqueue job  │
                       │  /api/stories       Proxy + TTL cache       │
                       │  /api/webhook       Stripe webhook          │
                       │  /api/inngest       Inngest handler         │
                       └──────┬──────────────┬────────────┬──────────┘
                              │              │            │
                              ▼              ▼            ▼
                    ┌──────────────┐  ┌──────────┐  ┌───────────────────────┐
                    │   Stripe     │  │ Inngest  │  │       Supabase        │
                    │  Checkout    │  │          │  │  Auth │ PostgreSQL    │
                    │  Webhooks    │  │ • Queues │  │  Realtime │ pg_cron   │
                    └──────────────┘  │ • Runs   │  └──────────┬────────────┘
                                      │ • Retries│             │
                                      │ • Cron   │             │ (reads/writes)
                                      └────┬─────┘             │
                                           │ calls             │
                                           ▼                   │
                                    ┌──────────────────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────┐
                       │        HikerAPI         │
                       │  Followers / Following  │
                       │  Profile info / Stories │
                       └─────────────────────────┘

                       ┌─────────────────────────┐
                       │         Resend          │
                       │  (called by Inngest)    │
                       └─────────────────────────┘
```

---

## Data Model

### `users`
Managed by Supabase Auth. Extended with app-level fields.

```sql
users (
  id            uuid PRIMARY KEY,   -- Supabase auth user id
  email         text UNIQUE,
  created_at    timestamptz
)
```

### `plans`
Seeded data. Not user-editable.

```sql
plans (
  id                        serial PRIMARY KEY,
  name                      text,
  max_targets               int,
  max_follower_count        int,        -- hard cap when adding target
  story_viewer_enabled      boolean,
  daily_sync_limit          int,        -- per user, across all targets
  cooldown_minutes          int,        -- between syncs of same target
  price_stripe_id           text        -- Stripe price ID
)
```

### `subscriptions`
One active subscription per user.

```sql
subscriptions (
  id                  uuid PRIMARY KEY,
  user_id             uuid REFERENCES users,
  plan_id             int REFERENCES plans,
  stripe_sub_id       text,
  status              text,            -- active | canceled | past_due
  current_period_end  timestamptz,
  created_at          timestamptz
)
```

### `pending_registrations`
Created on Stripe payment success. Expires after 24h.

```sql
pending_registrations (
  id          uuid PRIMARY KEY,
  email       text UNIQUE,
  plan_id     int REFERENCES plans,
  stripe_session_id text,
  expires_at  timestamptz,
  used        boolean DEFAULT false,
  created_at  timestamptz
)
```

### `targets`
Instagram accounts being monitored.

```sql
targets (
  id              uuid PRIMARY KEY,
  user_id         uuid REFERENCES users,
  ig_user_id      text NOT NULL,       -- stable pk from Instagram
  ig_username     text NOT NULL,       -- updated on sync if changed
  follower_count  int,
  status          text,                -- active | paused_private | paused_out_of_plan | paused_error | syncing
  added_at        timestamptz,
  last_synced_at  timestamptz
)
```

### `followers_state`
Current known follower set per target. Used as the "previous snapshot."

```sql
followers_state (
  target_id   uuid REFERENCES targets,
  ig_user_id  text,
  ig_username text,
  PRIMARY KEY (target_id, ig_user_id)
)
```

### `following_state`
Current known following set per target.

```sql
following_state (
  target_id   uuid REFERENCES targets,
  ig_user_id  text,
  ig_username text,
  PRIMARY KEY (target_id, ig_user_id)
)
```

### `events`
Immutable change log. 90-day retention via pg_cron cleanup.

```sql
events (
  id            uuid PRIMARY KEY,
  target_id     uuid REFERENCES targets,
  user_id       uuid REFERENCES users,
  event_type    text,    -- FOLLOWER_ADDED | FOLLOWER_REMOVED | FOLLOWING_ADDED | FOLLOWING_REMOVED
  ig_user_id    text,
  ig_username   text,
  detected_at   timestamptz,
  sync_run_id   uuid REFERENCES sync_runs
)
```

### `sync_runs`
One record per manual sync attempt.

```sql
sync_runs (
  id              uuid PRIMARY KEY,
  target_id       uuid REFERENCES targets,
  user_id         uuid REFERENCES users,
  status          text,        -- running | completed | failed
  request_units   int,
  started_at      timestamptz,
  completed_at    timestamptz,
  error_message   text
)
```

### `daily_stats`
Aggregated daily counts per target. Permanent retention.

```sql
daily_stats (
  target_id             uuid REFERENCES targets,
  date                  date,
  followers_gained      int DEFAULT 0,
  followers_lost        int DEFAULT 0,
  following_gained      int DEFAULT 0,
  following_lost        int DEFAULT 0,
  PRIMARY KEY (target_id, date)
)
```

### `request_unit_usage`
Cost tracking per user and sync.

```sql
request_unit_usage (
  id          uuid PRIMARY KEY,
  user_id     uuid REFERENCES users,
  sync_run_id uuid REFERENCES sync_runs,
  units       int,
  recorded_at timestamptz
)
```

### `email_log`
Prevent duplicate weekly emails.

```sql
email_log (
  id          uuid PRIMARY KEY,
  user_id     uuid REFERENCES users,
  week_start  date,
  sent_at     timestamptz,
  UNIQUE (user_id, week_start)
)
```

---

## API Routes (Next.js)

| Method | Route | Description |
|---|---|---|
| POST | `/api/webhook` | Stripe webhook (payment confirmed → create pending_registration) |
| POST | `/api/sync` | Validate request, create sync_run, call `inngest.send()`, return 202 |
| GET | `/api/stories/[targetId]` | Proxy story media from HikerAPI with TTL Cache-Control headers |
| POST | `/api/inngest` | Inngest handler — receives and executes all queued job events |

All other data access uses Supabase client directly from the frontend (with Row Level Security enforcing user isolation).

---

## Row Level Security Policy (Supabase)

All tables with `user_id` enforce:

```sql
-- Users can only read/write their own rows
CREATE POLICY "user_isolation" ON targets
  USING (user_id = auth.uid());
```

This eliminates the need for a custom authorization layer.

---

## Auth & Payment Flow

```
1. User selects plan → Stripe Checkout (email required)
2. Stripe calls POST /api/webhook (payment_intent.succeeded)
3. Webhook creates pending_registrations row (expires_at = now + 24h)
4. Redirect to /register?email=...
5. User sets password → Supabase Auth creates user
6. Registration handler checks pending_registrations for email:
   - Not found or expired → reject
   - Found → create subscription row, mark pending as used
7. User lands on /dashboard
```

---

## Sync Flow

The sync is **asynchronous**. The HTTP request returns immediately — Inngest executes the job in the background with no timeout limit. The frontend receives results via Supabase Realtime when the job completes.

```
User clicks "Sync"
    │
    ▼
POST /api/sync { targetId }                     ← HTTP request opens
    │
    ├── Verify target belongs to user (RLS)
    ├── Check cooldown: last_synced_at + cooldown > now? → 429
    ├── Check daily limit: syncs_today >= plan.daily_sync_limit? → 429
    ├── Acquire concurrent sync lock:
    │     SELECT id FROM targets WHERE id = ? AND status = 'active'
    │     FOR UPDATE SKIP LOCKED
    │     → row locked? → 409
    ├── Set target.status = 'syncing'
    ├── Create sync_run (status = 'queued')
    ├── inngest.send('sync/run', { syncRunId, targetId, planConfig })
    └── Return 202 { syncRunId }                ← HTTP request closes in < 500ms

Frontend subscribes to Supabase Realtime on sync_runs WHERE id = syncRunId
    │
    ▼
Inngest receives event → executes sync function (no timeout limit)
    │
    ├── Update sync_run (status = 'running')
    │
    ├── FOR each page up to plan.page_cap:
    │     ├── HikerAPI: fetch followers page
    │     │   (Inngest built-in rate limit: max 20 req/s across all concurrent jobs)
    │     │   (auto retry with backoff on 429 — declared in Inngest function config)
    │     └── Accumulate results
    │
    ├── HikerAPI: fetch following (same pattern)
    │
    ├── Diff against followers_state:
    │     added   = current - state → INSERT into followers_state
    │     removed = state - current → DELETE from followers_state WHERE ig_user_id IN (removed)
    │     (same for following_state)
    │
    ├── INSERT events for added/removed
    ├── UPSERT daily_stats for today
    ├── INSERT request_unit_usage
    ├── UPDATE sync_run (status = 'completed', request_units = N)
    └── UPDATE target (status = 'active', last_synced_at = now(), follower_count = N)
             │
             ▼
    Supabase Realtime pushes sync_run update to frontend
    Frontend displays diff results
```

### Inngest Function Config (declared once in code)

```typescript
inngest.createFunction(
  {
    id: 'sync-target',
    rateLimit: {
      limit: 20,
      period: '1s',
      key: 'event.data.planConfig.hikerApiKey'  // shared across all jobs
    },
    retries: 3,
    concurrency: { limit: 50 }  // max 50 sync jobs running simultaneously
  },
  { event: 'sync/run' },
  async ({ event, step }) => { /* sync logic here */ }
)
```

### Auto Sync (enabled per plan)

When auto sync is enabled on a plan, one additional Inngest scheduled function fires on the plan's configured interval:

```typescript
inngest.createFunction(
  { id: 'auto-sync-scheduler', concurrency: { limit: 1 } },
  { cron: '*/15 * * * *' },  // every 15 minutes
  async ({ step }) => {
    // fetch all active targets eligible for sync (cooldown elapsed)
    const targets = await step.run('fetch-eligible', () => getEligibleTargets())

    // fan-out: one sync/run event per target
    // Inngest queues and drains at controlled concurrency
    await inngest.sendMany(targets.map(t => ({ name: 'sync/run', data: t })))
  }
)
```

500 concurrent auto syncs do not explode — Inngest queues them and drains at the declared concurrency limit.

---

## Story Viewer Flow

```
Client requests GET /api/stories/[targetId]
    │
    ├── Verify user session
    ├── Verify plan allows story_viewer
    ├── Verify target belongs to user
    │
    ▼
HikerAPI: fetch story items for ig_user_id
    │
    ▼
Return story metadata (URLs proxied through /api/stories/media?url=...)
    │
(Client renders stories inline — media never stored)
```

---

## Weekly Email (Inngest Scheduled Function)

Replaced pg_cron HTTP call with a native Inngest scheduled function. Cleaner code, visible in Inngest dashboard, retries on failure.

```typescript
inngest.createFunction(
  { id: 'weekly-email', retries: 2 },
  { cron: '0 9 * * 1' },   // every Monday 09:00 UTC
  async ({ step }) => {
    const users = await step.run('fetch-eligible-users', () =>
      getUsersActiveThisWeekWithActivity()
    )
    for (const user of users) {
      await step.run(`send-email-${user.id}`, () =>
        sendWeeklyEmail(user)   // calls Resend, inserts email_log row
      )
    }
  }
)
```

Handler logic (unchanged):
1. Find users who logged in during past 7 days AND have activity_count > 0
2. Skip if `email_log` row exists for (user_id, week_start) — idempotent
3. Build per-target summary from daily_stats + events
4. Send via Resend
5. Insert email_log row

> **pg_cron is kept** only for the 90-day event cleanup job — a lightweight SQL DELETE that doesn't need Inngest's overhead.

---

## Scale Considerations (1000 Users)

The architecture supports 1000 customers with three changes from the free-tier starting point:

### Load Profile at 1000 Users

| Metric | Value | Basis |
|---|---|---|
| Daily active users | ~200 (20%) | Typical SaaS DAU |
| Avg targets per user | 2 | Mix of plan tiers |
| Daily syncs | ~600 | 200 users × 3 syncs/day |
| Peak concurrent syncs | 10–15 | Spread across peak hour |
| followers_state rows | ~40M | 1000 × 2 targets × 20k avg followers |
| followers_state storage | ~3.2GB | 40M rows × 80 bytes |

### Changes Required at 1000 Users

**1. Supabase Free → Pro ($25/month)**
Free tier storage is 500MB — followers_state alone requires ~3.2GB at 1000 users. Pro gives 8GB storage and higher connection pool limits.

**2. Resend Free → Paid ($20/month)**
Weekly emails to 1000 users = ~4,300/month. Free tier covers 3,000/month.

**3. Diff-only writes to followers_state (already in design)**
DELETE all + INSERT all = 40k row ops per sync. Diff-only = ~200 row ops per sync. 200× reduction in write amplification — critical at this scale.

**4. HikerAPI Rate Limiting — Handled by Inngest (already in design)**
Inngest's built-in rate limiting across all concurrent jobs replaces the previous `api_rate_state` DB-row workaround. Declared once in the function config.

### Auto Sync at 1000 Users — Now Supported by Inngest

With Inngest in the stack, auto sync at 1000 users works cleanly:
- Inngest scheduled function fires every 15 min
- Fans out one `sync/run` event per eligible target
- Inngest queues and drains at declared concurrency limit (e.g. 50 at a time)
- HikerAPI rate limit enforced globally via Inngest rate limiting config
- No infrastructure changes needed — it's a feature toggle in code

**Auto sync cost at 1000 users** depends on sync frequency (see Inngest pricing section).

### Monthly Infrastructure Cost at Scale

| Service | At Launch | At 1000 Users (manual sync) | At 1000 Users (auto sync, 4h interval) |
|---|---|---|---|
| Vercel | $0 | $0 | $0 |
| Supabase | $0 | $25 | $25 |
| Inngest | $0 | $0 (18k runs/mo, free tier) | ~$52 (360k runs/mo) |
| Resend | $0 | $20 | $20 |
| Stripe | % per txn | % per txn | % per txn |
| HikerAPI | Variable | Variable | Variable (higher) |
| **Fixed monthly** | **$0** | **$45** | **~$97** |

---

## Security Considerations

- HikerAPI key: server-side only (Inngest functions + Next.js API routes), never in client bundle
- Stripe webhook: verified via `stripe.webhooks.constructEvent()` with signing secret
- Inngest webhook (`/api/inngest`): verified via Inngest signing key — rejects any requests not from Inngest
- Supabase RLS: all tables with `user_id` are isolated per authenticated user — enforced at DB level
- Concurrent sync guard: `FOR UPDATE SKIP LOCKED` prevents race condition when two requests target the same row simultaneously
- Story media proxy: authenticated endpoint — no public media URLs exposed to client
