# Product Brief: Instagram Follower Monitoring SaaS

## Overview

A subscription-based web application that lets users monitor **public Instagram accounts** — tracking exactly who followed and unfollowed each monitored account after every manual sync. Users see usernames, can view public stories anonymously, and receive a weekly activity summary email.

The product is built to maximise value within a fixed development budget by composing existing managed services rather than building infrastructure from scratch. Every layer — scraping, queuing, auth, payments, email — is handled by a specialised provider.

---

## Core Features

### 1. Target Management

Users add a public Instagram account by username. On add, we validate via HikerAPI:
- Resolve username → stable Instagram user ID (`pk`)
- Reject if account is private
- Reject if follower count exceeds the plan's limit
- Reject if user is at their max target count for the plan

Stable `ig_user_id` is stored as the primary reference. `ig_username` is updated on each sync if the username has changed.

---

### 2. Follower/Following Change Detection

On every manual sync, the system:
1. Fetches the full current followers and following lists from HikerAPI (paginated, up to the plan's page cap)
2. Diffs against the last known state stored in the database
3. Writes only the changed rows (`FOLLOWER_ADDED`, `FOLLOWER_REMOVED`, `FOLLOWING_ADDED`, `FOLLOWING_REMOVED`)
4. Updates the activity feed, daily stats, and request unit usage records

Uses a **diff-only write algorithm** — not delete-all + insert-all. At 1,000 users this is ~266× fewer database operations.

---

### 3. Activity Feed & Alerts

- **Immediate in-app:** Results appear on screen the moment the sync job completes via Supabase Realtime (WebSocket push — no polling)
- **Weekly email:** Every Monday, conditional on the user having logged in during the past 7 days AND having activity > 0 for the week. Sent via Resend.

---

### 4. Anonymous Story Viewing

Plan-gated feature. Stories fetched server-side via HikerAPI — the user's browser never contacts Instagram directly. Media proxied through `/api/stories`. Not stored permanently. Cached at Vercel's edge using a dynamic TTL calculated from each story's `taken_at` timestamp (stories expire 24 hours after posting on Instagram).

---

### 5. Subscription Plans & Enforcement

Each plan defines:
- `max_targets` — max monitored accounts
- `max_follower_count` — reject accounts above this threshold on add
- `story_viewer_enabled` — gates the story viewer
- `daily_sync_limit` — max syncs per user per calendar day
- `cooldown_minutes` — minimum time between syncs of the same target
- `page_cap` — max pages fetched per sync (controls HikerAPI cost)
- `max_units_per_sync` — hard cost ceiling per sync job

Enforcement at every relevant point: target add, sync start, mid-sync, and story viewer access.

---

## Out of Scope (MVP)

| Item | Why |
|---|---|
| Custom scraper (Puppeteer/Playwright) | Replaced by HikerAPI — same outcome, fraction of the build cost |
| Mobile push notifications (FCM/APNs) | Requires a mobile app; replaced by Realtime + weekly email |
| Facebook monitoring | Not required |
| Mobile app | Web only |
| Admin/metrics dashboard | Not required |
| AI monitoring recommendations | Nice-to-have, not in budget |
| Automatic background syncing | Deferred — infrastructure-ready via Inngest, must be priced into plans first |
| Accounts with >1M followers | Post-MVP — requires architectural adjustment |
| Landing page | Client already has it |

---

## User Journey

```
Client's existing landing page (domain.com)
    ↓  "Get Started" → app.domain.com/plans
Plan selection page (we build this, styled to match client branding)
    ↓  user selects plan
Stripe Checkout (payment required before account creation)
    ↓  payment confirmed
Stripe webhook → create pending_registrations record → send registration email (7-day token link)
    ↓  user clicks email link
/register?token=uuid (email pre-filled read-only, user sets password)
    ↓  atomic: create Supabase Auth account + subscription + mark token used
/dashboard
```

**Recovery path:** If registration link expires (7 days), the register page offers a "resend link" option — no new charge required.

---

## Tech Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend + API | Next.js (App Router) on Vercel | Single repo, no separate backend, free hosting tier |
| Database | Supabase PostgreSQL | Managed, RLS built-in, free tier covers MVP |
| Auth | Supabase Auth | Email/password, session management, password reset |
| Async jobs | Inngest | No timeout, built-in rate limiting, retries, concurrency control, monitoring dashboard |
| Real-time sync status | Supabase Realtime | WebSocket push on sync completion — no polling |
| Scheduled cleanup | pg_cron | 90-day event cleanup only (lightweight SQL DELETE) |
| Instagram data | HikerAPI | Handles anti-bot, proxy rotation, pagination — server-side only |
| Payments | Stripe | Checkout, webhooks, subscription lifecycle |
| Email | Resend | Registration emails + weekly summaries |
| Story caching | Vercel Edge Cache | Dynamic TTL via Cache-Control headers — no extra infrastructure |

---

## Development Approach

Development proceeds in four phases using **BMAD v6** (AI-assisted development).

**Auth stub approach:** Instead of blocking on auth, development starts with a middleware stub that injects a hardcoded `DEV_USER_ID` from environment variables. All app code calls `getUser()` as normal. When real Supabase Auth is wired in (Phase 4), one file changes — no app-level code changes required.

### Phase 1 — Foundation
Project scaffold, Supabase schema (all tables + RLS), seed data, dummy user, all services wired locally.

### Phase 2 — Core Engine
Target management, sync engine (async Inngest flow, diff algorithm, Realtime push), plan enforcement, HikerAPI integration.

### Phase 3 — Dashboard + Remaining Features
Dashboard UI (targets, activity feed, stats), story viewer, weekly email, Stripe subscription webhooks, client branding, production deployment.

### Phase 4 — Auth (Separate Milestone, $275–$300)
Login page, registration page (post-payment token flow), Stripe webhook → pending_registrations, forgot password. Quoted separately because auth was explicitly listed as a client prerequisite in the original proposal.

---

## Budget

- **Fixed development cost:** $1,600 (Upwork milestone — ~40 developer hours with BMAD v6 efficiency)
- **AI model cost:** ~$100/month during development
- **Infrastructure at launch:** $0/month fixed (all free tiers)
- **Infrastructure at 1,000 users (manual sync):** ~$45/month fixed (Supabase Pro + Resend paid)
- **Infrastructure at 1,000 users (auto sync, 4h interval):** ~$97/month fixed

---

## Open Items Requiring Client Confirmation

| # | Item | Blocks |
|---|---|---|
| 1 | Follower count limit per plan tier | Target add validation, `plans` table seed |
| 2 | Cooldown duration per plan tier | Sync cooldown enforcement |
| 3 | Daily sync limit per plan tier | Sync limit enforcement |
| 4 | Page cap per plan tier | HikerAPI cost per sync |
| 5 | Timezone for weekly emails | Currently UTC |
| 6 | Branding assets (logo, colours, fonts) | Client-facing pages |
| 7 | HikerAPI account credentials | Phase 2 sync engine testing |
| 8 | Stripe account with products + prices configured | Phase 3 webhook testing |
| 9 | Resend account with verified sending domain | Phase 3 email testing |
| 10 | `app.domain.com` subdomain DNS → Vercel | Go-live |
