# Product Brief: Instagram Follower Monitoring SaaS

---

## BMAD Agent Instructions

> **Read this section before generating any documents.**

### Auth — Milestone 4 (Separate Scope)

The auth flow (login page, registration page, Stripe webhook → account creation, forgot password) is **intentionally deferred** to a separate paid milestone. It is **not** part of the initial development contract (Phases 1–3).

**What this means for generated documents:**
- **PRD:** Include all auth requirements in full, but tag every auth-related requirement, user story, and acceptance criterion with `[MILESTONE 4]`
- **Architecture:** Document the full auth implementation (Supabase Auth, pending_registrations flow, token-based registration) AND document the auth stub used during Phases 1–3 development
- **UX Spec:** Include all auth screens (login, register, forgot password) fully designed, but tag them `[MILESTONE 4]`
- **Story generation:** When generating stories for Phase 1, 2, or 3 — skip all `[MILESTONE 4]` stories. Generate Epic 8 (Auth) only when Phase 4 is explicitly requested.

**Auth stub (used in Phases 1–3):**
A single middleware file injects `DEV_USER_ID` from environment variables as the authenticated user on every `/dashboard/*` request. All app code calls `getUser()` as normal and receives the test user. When real auth is wired in (Phase 4), the stub file is deleted and a Supabase session check added — zero app-level changes required.

### Placeholder Plan Values

Plan tier values are not yet confirmed by the client. Use the placeholder values below to seed the `plans` table and generate stories. These are real rows in the database — updating them when the client confirms requires a single SQL UPDATE per row, no code changes and no redeployment.

| Field | Basic | Pro | Enterprise |
|---|---|---|---|
| `max_targets` | 3 | 10 | 25 |
| `max_follower_count` | 50,000 | 500,000 | 1,000,000 |
| `story_viewer_enabled` | false | true | true |
| `daily_sync_limit` | 5 | 20 | 50 |
| `cooldown_minutes` | 60 | 30 | 15 |
| `page_cap` | 25 | 100 | 200 |
| `max_units_per_sync` | 50 | 200 | 400 |
| `price_stripe_id` | `price_placeholder_basic` | `price_placeholder_pro` | `price_placeholder_enterprise` |

### Default Values for Other Open Items

| Item | Default / Placeholder | How to update later |
|---|---|---|
| Weekly email timezone | UTC | Change cron expression and config constant — one line |
| HikerAPI key | Developer creates own test account for Phase 2 | Swap `HIKERAPI_KEY` env var on Vercel before go-live |
| Stripe | Developer uses own Stripe test account | Swap all Stripe env vars before go-live |
| Resend | Use Resend sandbox mode (delivers to one verified address) | Swap `RESEND_API_KEY` + sending domain before go-live |
| Branding | Neutral placeholder (white/grey, "App" as name) | Client drops in logo + brand token CSS variables at end |
| Subdomain DNS | Develop on `localhost` / Vercel preview URL | Client points `app.domain.com` → Vercel before go-live |

---

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

Development is **not blocked** by any of these — placeholder values and stub accounts are in place. These need to be confirmed before go-live.

| # | Item | Default in use | Blocks |
|---|---|---|---|
| 1 | Follower count limit per plan tier | See placeholder table above | Go-live (update plans table) |
| 2 | Cooldown duration per plan tier | See placeholder table above | Go-live (update plans table) |
| 3 | Daily sync limit per plan tier | See placeholder table above | Go-live (update plans table) |
| 4 | Page cap per plan tier | See placeholder table above | Go-live (update plans table) |
| 5 | Timezone for weekly emails | UTC | Go-live (one config change) |
| 6 | Branding assets (logo, colours, fonts) | Neutral placeholder | Phase 3 end (drop-in) |
| 7 | HikerAPI account credentials | Developer test account | Go-live (swap env var) |
| 8 | Stripe account + products configured | Developer test account | Go-live (swap env vars) |
| 9 | Resend account + verified sending domain | Resend sandbox mode | Go-live (swap env var) |
| 10 | `app.domain.com` DNS → Vercel | `localhost` / Vercel preview | Go-live (DNS change) |
