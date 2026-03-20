# Product Requirements Document (PRD)

# Instagram Monitoring SaaS (Powered by HikerAPI)

---

## 1. Product Overview

### 1.1 Purpose

Build a subscription-based web application that enables users to monitor **public Instagram accounts** using manual sync operations.

The platform allows:

- Identifying exactly **who followed and who unfollowed** a monitored account
- Tracking follower growth and loss (count + user-level identity)
- Tracking following behavior changes
- Viewing public stories
- Receiving weekly activity reports (only if activity > 0)

The product enforces plan-based restrictions and usage limits.

**Primary Value Proposition:**
The core value of the system is enabling users to clearly see the usernames and profiles of accounts that followed or unfollowed their monitored targets after each manual sync.

---

## 2. Target Users

### Primary Users

1. Influencers tracking growth
2. Agencies monitoring clients
3. Brands monitoring competitors

### Secondary Users

- Growth analysts

---

## 3. User Journey

### Client-Mandated Flow

1. Landing Page
2. Plan Selection
3. Payment
4. Registration (Account creation) `[MILESTONE 4]`
5. Dashboard Access

### Important Notes

- Payment must be associated with email.
- If user abandons registration after payment, the registration token expires after a configurable duration (default: **7 days**). Expired links show a "Resend registration link" option — no new charge required.

---

## 4. Subscription & Plans

Each plan defines:

- `max_targets`
- `max_follower_count`
- `story_viewer_enabled`
- `daily_sync_limit`
- `cooldown_minutes`
- `page_cap` — max pages fetched per sync (controls HikerAPI cost and job duration)
- `max_units_per_sync` — hard request unit ceiling per sync (secondary cost safeguard)

### Placeholder Values (use until client confirms)

> These seed the `plans` table. Updating them requires a single SQL UPDATE — no code changes, no redeployment.

| Field                  | Basic  | Pro     | Enterprise |
| ---------------------- | ------ | ------- | ---------- |
| `max_targets`          | 3      | 10      | 25         |
| `max_follower_count`   | 50,000 | 500,000 | 1,000,000  |
| `story_viewer_enabled` | false  | true    | true       |
| `daily_sync_limit`     | 5      | 20      | 50         |
| `cooldown_minutes`     | 60     | 30      | 15         |
| `page_cap`             | 25     | 100     | 200        |
| `max_units_per_sync`   | 50     | 200     | 400        |

### Enforcement Rules

- Target cannot be added if `follower_count` exceeds plan threshold.
- If target later exceeds threshold → status = **"Paused – Out of Plan"**.
- Private accounts are rejected on add.
- If account becomes private → monitoring paused.

---

## 5. Sync Model

### Manual Sync Only (MVP)

There is **NO automatic background syncing** at MVP. Auto sync is architecturally ready via Inngest (fan-out scheduler already in the design) but is deferred until it can be priced into subscription plans — at 1,000 users with a 4-hour interval it costs ~$52/month in Inngest alone.

### Constraints

- Per-target cooldown (cannot sync same target within X minutes).
- Per-user daily sync limit (across all targets).
- Sync cannot run if another sync is in progress for same target.
- Sync jobs must be idempotent.

---

## 6. Core Features

### 6.1 Target Management

Add target by Instagram username.

#### Validation Steps

- Resolve username → obtain `pk` (igUserId).
- Reject if private.
- Reject if `follower_count` exceeds plan limit.

#### Storage

- `igUserId` (primary reference)
- `igUsername` (updateable if changed)

---

### 6.2 Monitoring Engine

Uses snapshot-diff model.

#### Sync Process

1. Fetch full followers list (paginated).
2. Fetch full following list (paginated).
3. Compute diffs:
    - `FOLLOWER_ADDED`
    - `FOLLOWER_REMOVED`
    - `FOLLOWING_ADDED`
    - `FOLLOWING_REMOVED`
4. Store events.
5. Store aggregated daily stats.
6. Record request unit usage.

---

### 6.3 Event Retention Policy

- Detailed event-level records retained for **90 days**.
- Daily aggregate statistics retained **indefinitely**.
- Snapshots are not stored long-term; state-based tables are used instead.

---

### 6.4 Story Viewer

Available only if plan allows (`story_viewer_enabled = true`).

#### Capabilities

- Fetch current public stories via HikerAPI (server-side only).
- Render media inside the app.
- Do not store media permanently — media passes through the proxy and is never written to database or storage.
- Media must be proxied through the backend (`/api/stories`). The browser never contacts Instagram directly.
- **Dynamic TTL edge caching:** Vercel edge cache stores the response with a `Cache-Control` header set to the story's remaining lifetime (calculated from `taken_at + 24h - now`). Multiple users viewing the same target share the cached response — HikerAPI is only called once per story per its remaining lifetime. Cache is ephemeral (evicts on TTL expiry or memory pressure).

---

### 6.5 Weekly Email

#### Conditions

Email is sent only if:

- User logged in during the week.
- AND activity count > 0.

#### Content

Per target:

- Followers gained/lost
- Following added/removed
- Link to dashboard

---

## 7. Cost Safeguards

System must track:

- Request units per sync
- Request units per user
- Average request units per plan tier

### Safeguards

- **Hard page cap per sync** (`plan.page_cap`): sync stops after this many HikerAPI pages regardless of remaining data; sync marked `completed_partial` if truncated.
- **Mid-sync unit budget** (`plan.max_units_per_sync`): cumulative request units are tracked per page fetch; sync stops early if units exceed the threshold before the page cap is reached.
- Daily sync cap (`plan.daily_sync_limit`).
- Cooldown enforcement (`plan.cooldown_minutes`).

### Metrics Tracked

- Cost per user
- Average syncs per user
- Churn rate

---

## 8. Data Model (High-Level)

- Users (Supabase Auth managed)
- Plans (seeded, not user-editable)
- Subscriptions (one active subscription per user)
- PendingRegistrations (created on Stripe payment; holds token for email-gated registration; expires after 7 days)
- Targets (monitored Instagram accounts)
- FollowersState (current known follower set per target — diff baseline)
- FollowingState (current known following set per target — diff baseline)
- Events (immutable change log — 90-day retention)
- DailyStats (aggregated daily counts — permanent)
- SyncRuns (one record per sync attempt)
- EmailLog (weekly email idempotency — prevents duplicate sends)
- RequestUnitUsage (HikerAPI cost tracking per sync)

---

## 9. Error Handling

### Target Statuses

- `active`
- `syncing`
- `paused_private` — account became private (detected during sync)
- `paused_out_of_plan` — follower count now exceeds plan limit (detected during sync)
- `paused_error` — non-transient HikerAPI error after retries exhausted

### Sync Run Statuses

- `queued` — created, waiting for Inngest to pick up
- `running` — Inngest executing
- `completed` — successful full sync
- `completed_partial` — sync stopped early (page cap or unit budget hit)
- `failed` — all retries exhausted

### Retry Logic

- Exponential backoff for transient API errors.
- No automatic infinite retries.

---

## 10. Non-Functional Requirements

- All HikerAPI calls must be server-side only.
- Access key must never be exposed.
- Rate limiting enforced.
- Full logging of sync jobs.
- Idempotent event creation.
- Secure authentication implementation.
- Webhook validation for billing.

---

## 11. Acceptance Criteria

- Paid user can register after payment.
- User can add valid public target within plan limits.
- Manual sync produces correct follower/following diff events.
- User can clearly view the list of usernames that followed and unfollowed each target after a sync.
- Sync limits enforced.
- Story viewer works for public accounts only.
- Weekly email sent only when conditions met.
- Out-of-plan targets automatically paused.
- API request unit usage tracked and logged.

---

## 12. Key Metrics

### Primary Metrics

- Cost per user
- Average syncs per user
- Churn

### Secondary Metrics

- Active targets per user
- Weekly login rate
- Sync success rate

---

## 13. Open Questions (Client Confirmation Required)

### Plan Configuration (blocks sync engine build)

1. Exact follower count threshold per plan tier?
2. Exact cooldown duration per plan tier?
3. Daily sync limit per plan tier?
4. Page cap per plan tier?
5. Max request units per sync per plan tier?

### Email

6. Timezone for weekly emails? (currently UTC — flag if user-local timezone required)

### Go-Live Accounts (client must create)

7. HikerAPI account — client creates; needed before Phase 2 sync engine testing
8. Stripe account with products and prices configured — needed before Phase 3 webhook testing
9. Resend account with verified sending domain — needed before Phase 3 email testing
10. `app.domain.com` subdomain DNS → Vercel — needed before go-live

### Branding

11. Logo, brand colours, and fonts for client-facing pages?

## 14. Development Approach

Development uses **BMAD v6** (AI-assisted development) in four phases:

1. **Phase 1 — Foundation:** Project scaffold, full DB schema + RLS, seed data, auth stub middleware, all services wired locally
2. **Phase 2 — Core Engine:** Target management, async sync engine, plan enforcement, HikerAPI integration
3. **Phase 3 — Dashboard + Features:** Dashboard UI, story viewer, weekly email, Stripe webhooks, client branding, production deployment
4. **Phase 4 — Auth `[MILESTONE 4 — SEPARATE SCOPE, $275–$300]`:** Login page, registration flow, Stripe webhook → pending_registrations, forgot password

**Auth stub (Phases 1–3):** A middleware stub injects `DEV_USER_ID` from environment variables as the authenticated user. All app code calls `getUser()` as normal. Swapping in real Supabase Auth (Phase 4) deletes the stub file and adds one Supabase session check — zero app-level changes required.

**Placeholder values:** Plan tier values (follower limits, cooldowns, sync limits, page caps) are seeded with placeholder values in Phase 1. When the client confirms the real numbers, a single `UPDATE plans SET ... WHERE name = ?` per row updates them — no code changes, no redeployment.

> **For story generation:** Generate stories for Phases 1–3 only. Skip all requirements and stories tagged `[MILESTONE 4]`. Generate Epic 8 (Auth) only when Phase 4 is explicitly requested.

---

**End of Document**
