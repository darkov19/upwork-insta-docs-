# Requirements & Solutions

## Instagram Monitoring SaaS

This document maps every client requirement — from the original product brief and the refined PRD — to exactly how we are solving it, which service handles it, and where we have made changes from the original ask and why.

---

## How to Read This Document

Each requirement section shows:

- **Requirement** — what the client asked for
- **Our Solution** — how we are building it
- **Service(s)** — which part of the stack handles it
- **Deviation** — if we changed the approach from what was originally requested, and why

---

## 1. Instagram Data Ingestion (Bot-Resistant)

**Requirement:**
Scrape public Instagram profile data — followers list, following list — without being detected. Handle anti-bot mechanisms, CAPTCHAs, fingerprinting, and rate limits. Use Puppeteer or Playwright with residential/mobile proxy rotation (Bright Data, Oxylabs).

**Our Solution:**
We do not build a custom scraper. We use **HikerAPI** — a managed API service that has already built and maintained all of this. We call their endpoints like any standard REST API and receive clean, structured data back. They handle proxy rotation, anti-bot measures, and rate limit management on their side.

**Service:** HikerAPI

**Deviation:** Yes — we replaced Puppeteer/Playwright + proxy infrastructure with HikerAPI.

Why: Building a reliable, bot-resistant scraper correctly requires 15–20 developer hours minimum, plus ongoing maintenance every time Instagram updates its detection. At the project budget, this would consume the majority of hours before a single product feature is built. HikerAPI achieves the same outcome — fetching public Instagram data without detection — at a fraction of the cost and maintenance burden. The end result for the client is identical: their users' monitored accounts are tracked accurately.

---

## 2. Follower & Following Change Detection

**Requirement:**
Detect who followed and who unfollowed a monitored account by comparing snapshots. Store data in MongoDB or PostgreSQL. Use a diffing algorithm to detect adds and removes. Support snapshot intervals of 5–15 minutes.

**Our Solution:**
We store the current known follower list for each monitored account in two PostgreSQL tables: `followers_state` and `following_state`. Each row represents one follower of one target account.

On every sync, the Inngest sync function:

1. Fetches the current followers list from HikerAPI (paginated, up to the plan's page cap)
2. Loads the existing `followers_state` from the database
3. Computes the difference in memory:
    - Accounts in the current list but not in the stored state → `FOLLOWER_ADDED`
    - Accounts in the stored state but not in the current list → `FOLLOWER_REMOVED`
4. Writes only the changed rows — inserts new followers, deletes removed ones
5. Stores each change as an event record with timestamp, username, and Instagram user ID
6. Same process repeated for the following list (`FOLLOWING_ADDED`, `FOLLOWING_REMOVED`)

Every follow and unfollow is recorded as an immutable event that the user can view in their activity feed.

**Service:** Supabase PostgreSQL + Inngest (`sync-target` function)

**Deviation:** Partial — on two points:

1. **MongoDB → PostgreSQL:** Our data is relational — targets belong to users, events belong to targets, states belong to targets. PostgreSQL handles this naturally with foreign keys and indexed queries. MongoDB is a document database better suited to unstructured data. PostgreSQL is the right tool for this data model.

2. **5–15 minute automatic intervals → Manual sync for MVP:** The PRD confirmed manual sync only. The client clicks a Sync button; nothing runs automatically unless the auto sync feature is enabled on their plan. Auto sync is supported by Inngest and can be enabled per plan — it is an infrastructure-ready feature, not a post-MVP rewrite.

Why manual sync for MVP: Automatic syncing every 5–15 minutes across 1,000 users and 2,000 targets generates ~360,000–1,000,000+ background jobs per month. This cost must be priced into subscription plans first. Starting with manual sync lets us launch within budget and enable auto sync as a paid plan feature once pricing is confirmed.

---

## 3. Activity Alerts

**Requirement:**
Push real-time alerts to users when follower changes are detected. Use Firebase Cloud Messaging (FCM) or Apple Push Notification service (APNs) with a background queue (Celery or BullMQ) managing thousands of concurrent notifications.

**Our Solution:**
We deliver activity alerts through two mechanisms:

**Mechanism 1 — Immediate in-app results:**
When a sync completes, the results appear on screen instantly via Supabase Realtime (a WebSocket connection). The user sees exactly who followed and who unfollowed, with usernames and profile links, the moment the sync job finishes. No refresh needed.

**Mechanism 2 — Weekly email summary:**
Every Monday, the Inngest `weekly-email` function sends a per-target summary of follower gains and losses to users who were active during the week and had activity. Delivered via Resend.

**Service:** Supabase Realtime (immediate results) + Inngest + Resend (weekly email)

**Deviation:** Yes — we replaced FCM/APNs push notifications with in-app Realtime results and weekly email.

Why: FCM and APNs are mobile push notification systems. They require a mobile app (iOS or Android) installed on the user's device. This is a web application — there is no mobile app. Browser push notifications exist as an alternative, but they require the user to grant browser notification permissions, have inconsistent cross-browser support, and add significant development complexity. The weekly email covers the "I wasn't looking at the app" use case. The Realtime in-app result covers the "I'm actively using the app right now" use case. Together they deliver the alerting value the client wants without mobile app infrastructure.

Background queue (Celery/BullMQ/Redis): Replaced by **Inngest**, which provides queuing, concurrency control, retries, and scheduling without requiring Redis infrastructure or a separate worker process to manage.

---

## 4. Anonymous Story Viewing

**Requirement:**
Allow users to view Instagram stories without the account owner seeing that their story was viewed. Fetch `.mp4` and `.jpg` media files through the backend. Use server-side proxying so the client browser is never attributed. No logged-in Instagram session used. Cache content via Cloudflare or Varnish for multi-user delivery.

**Our Solution:**
Story media is fetched through our `/api/stories` API route on the server. The browser never makes a direct request to Instagram — it only talks to our server. Our server calls HikerAPI, which retrieves the story media without a logged-in session. The media URL returned by HikerAPI is served through our proxy, so Instagram sees only our server's request, not the end user's browser.

**Caching** is handled by Vercel's edge network. Instead of a fixed cache duration, we calculate the precise remaining lifetime of each story:

```
Story posted 8 hours ago → Instagram expires it in 16 hours → cache for 16 hours
Story posted 22 hours ago → Instagram expires it in 2 hours → cache for 2 hours
```

This means the cache automatically expires exactly when the story expires on Instagram. Multiple users viewing the same target's stories get served from the Vercel edge cache — HikerAPI is only called once per story per its remaining lifetime.

**Plan gate:** Story viewer is only available to users on plans where `story_viewer_enabled = true`. Users on plans without this feature see the option as locked.

**No permanent storage:** Story media is never written to our database or any storage service. It passes through the proxy and is cached temporarily at the edge. When the cache expires, the content is gone.

**Service:** Next.js `/api/stories` route + HikerAPI + Vercel Edge Cache

**Deviation:** Minor — on the caching layer only.

The original requirement asked for Cloudflare or Varnish. We use Vercel's built-in edge cache instead. This achieves identical behaviour — global CDN caching, multi-user delivery — with zero additional infrastructure. Our dynamic TTL approach is actually an improvement over a fixed cache duration because it never serves a story beyond its Instagram expiry.

---

## 5. User Journey — Payment Before Registration

**Requirement:**
The user flow must follow this exact sequence: Landing Page → Plan Selection → Payment → Registration → Dashboard. Payment must be tied to an email address. If a user abandons registration after paying, the session must expire after a configurable duration.

**Our Solution:**

```
domain.com (client's existing landing page)
    ↓  user clicks "Get Started"
app.domain.com/plans  (plan selection — we build this)
    ↓  user selects a plan
Stripe Checkout  (payment — Stripe hosted, we never touch card data)
    ↓  payment confirmed
Stripe calls our /api/webhook with signed notification
    ↓  we verify the signature and:
       1. create a pending_registrations record:
          { email, plan_id, token: randomUUID(), expires_at: now + 7 days }
       2. send a registration email via Resend:
          Subject: "Complete your account setup"
          Link:    app.domain.com/register?token=<uuid>
Customer receives email and clicks the link
    ↓
app.domain.com/register?token=<uuid>  (email pre-filled read-only)
    ↓  user sets a password
       we check: does a valid pending_registrations record exist for this token?
       valid (exists, not expired, not used) → create account, create subscription, set used_at = now()
       not found                             → reject
       already used                          → reject (account already exists — direct to /login)
       expired, not yet used                 → show "Link expired" + resend option (no new charge)
app.domain.com/dashboard
```

**Webhook timing:** Stripe fires the webhook and redirects the customer at roughly the same time. If the customer clicks the emailed link before the webhook has written the DB row, the register page shows "Confirming your payment…" and polls for up to 10 seconds before surfacing an error. In practice the webhook arrives within 1–2 seconds, so this is rarely visible.

**Recovery path (customer didn't register in time or lost the email):**
The registration link remains valid for 7 days. If it expires, the `/register?token=...` page shows an email input and a "Resend registration link" button. Submitting the email re-issues a new token and sends a new email — no new payment required. The original `pending_registrations` record (with `used_at = null`) is proof that payment was received.

The 7-day expiry is stored in the `pending_registrations` table as `expires_at`. The duration is configurable — just a value in the database.

**Service:** Next.js pages + Stripe + Supabase Auth + Supabase PostgreSQL + Resend

**Deviation:** Minor extension of the original requirement. The requirement specified expiry after a configurable duration — we honour that. We added a registration email (necessary for the customer to return after abandoning) and a token-based URL (necessary to make the link unguessable and re-issuable). The expiry window is extended from 24 hours to 7 days to avoid locking out legitimate paying customers who take a few days to set up their account.

---

## 6. Subscription Plans & Enforcement

**Requirement:**
Each plan defines: maximum number of targets, maximum follower count allowed per target, story viewer access, daily sync limit per user, and cooldown between syncs of the same target. Enforce these limits actively.

**Our Solution:**
All plan limits are stored in the `plans` table:

```
plans:
  max_targets               → max accounts a user can monitor
  max_follower_count        → reject accounts above this on add
  story_viewer_enabled      → gates the story viewer feature
  daily_sync_limit          → max syncs per user per calendar day
  cooldown_minutes          → minimum time between syncs of the same target
  page_cap                  → max pages fetched per sync (controls HikerAPI cost)
```

**Enforcement at every relevant point:**

| Action          | Enforcement                                               |
| --------------- | --------------------------------------------------------- |
| Adding a target | Reject if account is private                              |
| Adding a target | Reject if follower count exceeds `max_follower_count`     |
| Adding a target | Reject if user already has `max_targets` targets          |
| Clicking Sync   | Reject if `last_synced_at + cooldown_minutes > now`       |
| Clicking Sync   | Reject if syncs today `>= daily_sync_limit`               |
| Clicking Sync   | Reject if another sync is already running for this target |
| During sync     | Stop fetching after `page_cap` pages                      |
| During sync     | Pause target if it has become private                     |
| During sync     | Pause target if follower count now exceeds plan limit     |
| Viewing stories | Block if `story_viewer_enabled = false` on plan           |

**Service:** Supabase PostgreSQL (`plans` table) + Next.js API routes + Inngest (`sync-target` function)

**Deviation:** None. All fields from the PRD are implemented.

**Open items still needing client input:**

- Exact follower count thresholds per tier
- Exact cooldown duration per tier
- Exact daily sync limit per tier
- Exact page cap per tier

---

## 7. Target Management

**Requirement:**
Users add a monitored account by Instagram username. Validate: resolve username to a stable Instagram user ID (`pk`), reject if private, reject if follower count exceeds plan limit. Store the stable user ID as the primary reference and the username as updateable.

**Our Solution:**
When a user types a username and clicks Add:

1. We call HikerAPI to resolve the username — this returns the stable `ig_user_id` (`pk`), current follower count, and public/private status
2. If private → reject with "This account is private and cannot be monitored"
3. If follower count exceeds `plan.max_follower_count` → reject with "This account exceeds your plan's follower limit"
4. If user already has `max_targets` targets → reject with plan upgrade prompt
5. All checks pass → save target with both `ig_user_id` and `ig_username`

On every subsequent sync, if the username has changed (Instagram allows username changes), we update `ig_username` to the current value. The `ig_user_id` never changes — it is Instagram's permanent internal ID for the account.

**Service:** Next.js API route + HikerAPI + Supabase PostgreSQL

**Deviation:** None.

---

## 8. Sync Constraints — Cooldown, Daily Limit, Concurrent Guard, Idempotency

**Requirement:**
Per-target cooldown (cannot sync the same target within X minutes). Per-user daily sync limit across all targets. Sync cannot run if another sync is already in progress for the same target. Sync jobs must be idempotent (running the same job twice produces the same result).

**Our Solution:**

**Cooldown:** Before starting a sync, we check `last_synced_at` against the plan's `cooldown_minutes`. If the cooldown has not elapsed, we return a 429 error with the time remaining.

**Daily limit:** We count `sync_runs` for the user where `started_at >= today_start_utc` and `status IN ('running', 'completed')`. If the count equals or exceeds `plan.daily_sync_limit`, we return a 429 error.

**Concurrent sync guard (race condition safe):** We use PostgreSQL's `FOR UPDATE SKIP LOCKED` to atomically claim the target row before proceeding. If another sync has already claimed the row, our query returns nothing and we return a 409 error. This is a database-level lock — even if two sync requests arrive at exactly the same millisecond, only one can claim the row. The other is cleanly rejected.

```sql
SELECT id FROM targets
WHERE id = ? AND status = 'active'
FOR UPDATE SKIP LOCKED
-- Returns the row to the first request only
-- The second request gets nothing → returns 409
```

**Idempotency:** Every sync creates a `sync_run` record with a unique ID. The Inngest function checks at the start whether this `sync_run` is already `completed`. If it is, it exits immediately without doing any work. Events are also guarded — we use the `sync_run_id` as a reference so the same event cannot be inserted twice for the same sync.

**Service:** Next.js `/api/sync` route + Supabase PostgreSQL + Inngest

**Deviation:** None.

---

## 9. Event Storage & Retention

**Requirement:**
Detailed event records (who followed, who unfollowed, with timestamps) retained for 90 days. Daily aggregate statistics retained indefinitely. Snapshots not stored long-term — use state-based tables instead.

**Our Solution:**

**Events table:** Every detected change is an immutable row:

```
{ target_id, user_id, event_type, ig_user_id, ig_username, detected_at, sync_run_id }
```

Event types: `FOLLOWER_ADDED`, `FOLLOWER_REMOVED`, `FOLLOWING_ADDED`, `FOLLOWING_REMOVED`

A pg_cron job runs every night at 2am UTC and deletes events older than 90 days:

```sql
DELETE FROM events WHERE detected_at < now() - INTERVAL '90 days';
```

**Daily stats table:** One row per target per day, storing follower/following gain and loss counts. Never deleted. Used for charts, trends, and weekly email summaries.

**State tables:** `followers_state` and `following_state` hold only the current known follower list for each target — not snapshots. They are updated on every sync using diff-only writes (only changed rows are inserted or deleted). This keeps storage lean regardless of how many times a target has been synced.

**Service:** Supabase PostgreSQL + pg_cron

**Deviation:** None.

---

## 10. Cost Safeguards

**Requirement:**
Track request units per sync, per user, and per plan tier. Enforce a hard page cap per sync. Stop sync if request unit usage exceeds a threshold. Enforce daily sync caps and cooldowns.

**Our Solution:**

**Tracking:** Every HikerAPI page response includes the number of request units consumed. The Inngest sync function accumulates this count and at the end of the sync writes a `request_unit_usage` record:

```
{ user_id, sync_run_id, units, recorded_at }
```

This gives us a complete cost history per user, per sync, and per plan tier.

**Hard page cap:** Each plan has a `page_cap` value. The sync function stops fetching after this many pages regardless of whether the full follower list has been retrieved. The sync is marked `completed_partial` if truncated.

**Unit budget mid-sync:** In addition to the page cap, we track cumulative units consumed during the sync. If units exceed a configurable threshold before the page cap is reached, the sync stops early. This is the safety net for unusually expensive pages.

**Daily and cooldown caps:** Enforced before the sync starts (see Requirement 8). These prevent excessive usage at both the per-target and per-user level.

**Inngest rate limiting:** Globally limits HikerAPI calls to 20 requests per second across all concurrent sync jobs — preventing runaway API costs from simultaneous heavy users.

**Service:** Inngest (`sync-target` function) + Supabase PostgreSQL (`request_unit_usage`, `plans` tables)

**Deviation:** None. All four safeguards from the PRD are implemented.

---

## 11. Target Status Management

**Requirement:**
Targets must have statuses: Active, Paused – Private, Paused – Out of Plan, Paused – Error, Syncing.

**Our Solution:**
The `targets` table has a `status` field. It moves through the following states:

```
ADDING     → validating on add (private? over limit?)
    ↓ passes
ACTIVE     → ready to sync
    ↓ user clicks Sync
SYNCING    → sync in progress (locked, no duplicate syncs)
    ↓ completes
ACTIVE     → back to normal

SYNCING → PAUSED_PRIVATE      (account became private during sync)
SYNCING → PAUSED_OUT_OF_PLAN  (follower count now exceeds plan limit)
SYNCING → PAUSED_ERROR        (HikerAPI returned a non-recoverable error)
```

Status changes happen inside the Inngest sync function, not in a background polling job. The next manual sync is when we discover that an account's status has changed.

When a target is paused, the user sees a clear status message in the dashboard explaining why — "Account is now private", "Account exceeds your plan's follower limit" — with an action they can take (upgrade plan, remove target).

**Service:** Supabase PostgreSQL + Inngest (`sync-target` function) + Next.js dashboard

**Deviation:** None.

---

## 12. Error Handling & Retries

**Requirement:**
Exponential backoff for transient API errors. No automatic infinite retries.

**Our Solution:**
Retry behaviour is declared in the Inngest function configuration:

```typescript
{
  retries: {
    attempts: 3,           // maximum 3 attempts (not infinite)
    backoff: 'exponential' // 1s → 2s → 4s between attempts
  }
}
```

For HikerAPI 429 (rate limit) responses specifically, Inngest also applies the jitter delay before each page fetch to reduce collision with other concurrent jobs.

If all 3 attempts fail, the sync is marked `failed` in `sync_runs`, the target is set to `paused_error`, and Supabase Realtime pushes the failure status to the user's browser immediately.

**Service:** Inngest (built-in retry config)

**Deviation:** None. The behaviour matches the requirement exactly — exponential backoff, fixed maximum attempts.

---

## 13. Non-Functional Requirements

### HikerAPI key never exposed to client

HikerAPI is only called from inside Inngest functions and Next.js API routes — server-side code only. The API key is stored as an environment variable on Vercel and Inngest. It is never included in any code that runs in the user's browser.

### Rate limiting enforced

Inngest enforces a global 20 requests/second limit across all concurrent HikerAPI calls. Plan-level page caps and daily sync limits provide additional layers of rate control.

### Full logging of sync jobs

Every sync creates a `sync_runs` record with: status, start time, completion time, request units consumed, and error message if applicable. This is the permanent audit log of all sync activity.

### Idempotent event creation

Events reference their `sync_run_id`. If the same sync function executes twice (Inngest retry after crash), the second run detects the completed `sync_run` record and exits without writing duplicate events.

### Secure authentication

Supabase Auth handles all authentication. Passwords are hashed and never stored in plain text. Session tokens are short-lived JWTs. Row Level Security in the database enforces that authenticated users can only access their own data.

All server-side auth checks use `supabase.auth.getUser()`, not `getSession()`. `getSession()` trusts the cookie without revalidating — an attacker can forge a cookie payload and bypass auth. `getUser()` makes a live network call to verify the token.

RLS must be explicitly enabled per table in Supabase (off by default). RLS policies use only `auth.uid()` — never `auth.jwt() -> 'user_metadata'`, which is writable by the authenticated user. The Supabase Admin client (service role key) is used only in server-only files and never bundled into the client.

### Webhook validation for billing

The Stripe webhook endpoint (`/api/webhook`) verifies every incoming request using Stripe's signing secret via `stripe.webhooks.constructEvent()` against the **raw request body** (not the parsed JSON — Next.js App Router must not parse the body before signature verification, or the bytes will differ and verification always fails).

The handler returns HTTP 200 immediately after DB insert and sends the registration email asynchronously. This prevents Stripe from treating a slow Resend API call as a timeout, marking the webhook failed, and retrying — which would trigger duplicate processing.

DB-level idempotency is enforced via a `UNIQUE` constraint on `stripe_event_id` with `INSERT ... ON CONFLICT DO NOTHING`. Stripe explicitly documents that the same event can be delivered more than once, and Vercel's serverless model means two concurrent invocations can both pass an application-level check before either one commits to the database.

The handler also processes `charge.refunded` and `charge.dispute.created` events to suspend user access, preventing a refunded or disputed customer from continuing to use the product.

**Service:** Vercel environment variables + Inngest + Supabase Auth + Supabase RLS + Stripe SDK

**Deviation:** None — strengthened with production-hardening details not in the original requirement.

---

## 14. Weekly Email

**Requirement:**
Send a weekly email only if: the user logged in during the past 7 days AND activity count is greater than zero. Content: per-target summary of followers gained/lost and following added/removed, with a link to the dashboard.

**Our Solution:**
The Inngest `weekly-email` function runs every Monday at 9am UTC. For each user it:

1. Checks `last_sign_in_at` from Supabase Auth — if more than 7 days ago, skip
2. Checks `email_log` — if an email was already sent for this week, skip (idempotent)
3. Queries `daily_stats` for the past 7 days across all the user's targets
4. If total activity is zero, skip
5. Builds a per-target summary and sends via Resend
6. Inserts an `email_log` record to prevent duplicates

The `email_log` insert at the end means that if the cron fires twice in the same week, the second run finds the existing record and skips — no duplicate emails.

**Service:** Inngest (`weekly-email` function) + Resend + Supabase PostgreSQL

**Deviation:** None.

---

## 15. Acceptance Criteria Checklist

| Acceptance Criterion                                          | How It Is Met                                                                                                                                               |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Paid user can register after payment                          | Stripe webhook creates `pending_registrations` record + sends registration email with token link; registration page validates token before creating account |
| Paid user who abandoned registration can recover              | Token link in email valid for 7 days; expired link shows resend option — no new charge required                                                             |
| Refunded or disputed customer loses access                    | `charge.refunded` and `charge.dispute.created` webhook events suspend subscription; access revoked at middleware                                            |
| Duplicate webhook delivery does not create duplicate accounts | `UNIQUE (stripe_event_id)` + `ON CONFLICT DO NOTHING` — DB-level idempotency                                                                                |
| User can add valid public target within plan limits           | HikerAPI validates on add: public check, follower count check, max targets check                                                                            |
| Manual sync produces correct follower/following diff events   | Inngest sync function: fetch → diff → insert FOLLOWER_ADDED/REMOVED, FOLLOWING_ADDED/REMOVED events                                                         |
| User can clearly view who followed and unfollowed after sync  | Activity feed in dashboard reads from `events` table, shows username and profile link per event                                                             |
| Sync limits enforced                                          | Cooldown, daily limit, and concurrent guard all checked before sync starts                                                                                  |
| Story viewer works for public accounts only                   | HikerAPI only fetches public account stories; plan gate enforced at API route level                                                                         |
| Weekly email sent only when conditions met                    | Inngest function checks: active last 7 days AND activity > 0 AND not already sent this week                                                                 |
| Out-of-plan targets automatically paused                      | Detected during sync when follower count exceeds plan limit; status set to `paused_out_of_plan`                                                             |
| API request unit usage tracked and logged                     | `request_unit_usage` table populated after every sync with units consumed                                                                                   |

---

## 16. What We Confirmed as Out of Scope

| Item                                 | Status       | Reason                                                                      |
| ------------------------------------ | ------------ | --------------------------------------------------------------------------- |
| Facebook monitoring                  | Out of scope | Client confirmed not required                                               |
| Mobile app                           | Out of scope | Web only                                                                    |
| FCM / APNs push notifications        | Out of scope | Requires mobile app; replaced by Realtime + weekly email                    |
| Admin / metrics dashboard            | Out of scope | Client confirmed not required                                               |
| AI agent for monitoring suggestions  | Out of scope | Nice-to-have, not in budget                                                 |
| Visual timeline of follows/unfollows | Out of scope | Activity feed covers this adequately for MVP                                |
| Auto sync (short interval)           | Deferred     | Infrastructure ready via Inngest; must be priced into plans before enabling |
| Accounts with >1M followers          | Deferred     | Requires different architecture (long-running worker); post-MVP             |

---

## 17. Open Items Pending Client Input

These must be confirmed before the sync engine is built:

| #   | Question                                     | Why It Blocks                                                            |
| --- | -------------------------------------------- | ------------------------------------------------------------------------ |
| 1   | Exact follower count threshold per plan tier | Seeds the `plans` table; enforcement logic depends on it                 |
| 2   | Cooldown duration per plan tier              | Core sync constraint                                                     |
| 3   | Daily sync limit per plan tier               | Core sync constraint                                                     |
| 4   | Page cap per plan tier                       | Controls HikerAPI cost and Inngest job duration                          |
| 5   | Timezone for weekly emails                   | Currently UTC; if user-local timezone is required, additional complexity |
