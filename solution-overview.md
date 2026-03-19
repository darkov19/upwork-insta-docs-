# Solution Overview

## Instagram Follower Monitoring SaaS

---

## What We're Building

A subscription-based web application that lets users monitor **public Instagram accounts** — tracking who followed and unfollowed them after each manual sync. Built to maximize value within a fixed budget by composing existing managed services rather than building infrastructure from scratch.

---

## The Lean Principle

> Buy, don't build. Every hour spent on infrastructure is an hour not spent on product.

### What We Are NOT Building

| Removed | Replaced By | Why |
|---|---|---|
| Custom Instagram scraper | HikerAPI | Handles anti-bot, proxies, rate limits for us |
| Custom auth system | Supabase Auth | Production-ready, free tier |
| Backend server (Node/Python) | Supabase Edge Functions + Next.js API routes | No server to manage or pay for |
| Job queue (BullMQ/Celery/Redis) | Inngest | Managed background jobs — no timeout, built-in rate limiting, retries, concurrency control, monitoring dashboard |
| Custom proxy infrastructure | HikerAPI includes it | Already priced in |
| Separate database hosting | Supabase PostgreSQL | Managed, free tier covers MVP |
| Push notification system | Weekly email only | Simpler, still valuable, no FCM/APNs complexity |
| CDN infrastructure (Cloudflare/Varnish) | Vercel Edge Cache | Story TTL caching handled via Cache-Control headers, zero infrastructure |
| Landing page | Client already has it | We integrate with their existing landing page |
| Admin/metrics panel | Out of scope | Not required for MVP |
| Facebook monitoring | Out of scope | Not required |

---

## What We Are Building

One Next.js app on `app.domain.com`. The client's landing page remains on `domain.com` untouched — CTA buttons there simply link to our app.

```
domain.com  (client owns — not our responsibility)
    │  CTA links: "Get Started" → app.domain.com/plans
    │             "Login"       → app.domain.com/login
    ▼
app.domain.com  (our Next.js app on Vercel)
  ├── /plans               Plan selection + Stripe Checkout
  ├── /register            Post-payment registration
  ├── /login               Login
  ├── /dashboard           Main app (targets, sync, activity feed)
  ├── /dashboard/stories   Story viewer
  └── API Routes
        ├── /api/sync       → validate + inngest.send('sync/run')
        ├── /api/inngest    → Inngest job handler (all background functions)
        ├── /api/stories    → proxy + TTL Cache-Control headers
        └── /api/webhook    → Stripe webhook handler
              │
              ▼
        Inngest (background jobs)
          ├── sync-target function (no timeout, rate limited, retried)
          ├── auto-sync-scheduler (cron, fans out to sync-target)
          └── weekly-email function (cron, calls Resend)
              │
              ▼
        Supabase
          ├── Auth (email/password)
          ├── PostgreSQL (all data)
          ├── Realtime (pushes sync status to frontend)
          └── pg_cron (90-day event cleanup only)
              │
              ▼
        HikerAPI (Instagram data)
        Stripe (billing)
        Resend (transactional email, called by Inngest)
```

> **Branding:** Client provides logo, colours, and fonts. Our app pages are styled to match their landing page — no migration required.

---

## Key Integrations

### HikerAPI
- Fetches followers list (paginated), following list, profile info, public stories
- Server-side only — API key never exposed to client
- Handles anti-bot mechanics, proxy rotation, rate limiting on their side
- Cost is per-request-unit; our plan limits control spend

### Inngest
- Executes sync jobs asynchronously with no timeout limit
- Built-in HikerAPI rate limiting (20 req/s across all concurrent jobs), declared in function config
- Built-in retry with exponential backoff on failures and 429s
- Concurrency control (max 50 sync jobs simultaneously)
- Weekly email scheduled function (replaces pg_cron for this job)
- Auto sync fan-out when enabled: one scheduled trigger → one event per eligible target → Inngest queues and drains
- Monitoring dashboard — every job visible with full logs
- **Free tier covers manual sync at 1000 users** (18k runs/month, free tier is 50k)

### Supabase
- **Auth**: Email/password, session management
- **PostgreSQL**: All application data
- **Realtime**: Pushes sync_run status updates to frontend when Inngest job completes — no polling
- **pg_cron**: 90-day event cleanup (lightweight SQL DELETE, stays in Supabase)

### Stripe
- Checkout session tied to email (payment before registration)
- Webhook confirms payment → unlocks registration
- Subscription management (plan upgrades, cancellations)

### Resend
- **Registration email** — sent immediately on payment; contains the unique token link to complete account setup
- Weekly summary emails (conditional on activity)
- Transactional emails (receipts, plan change notifications)

### Vercel
- Next.js hosting, free tier covers MVP traffic
- Environment variable management for secrets
- **Edge Cache** for story media — Cache-Control headers set dynamically based on story `taken_at` timestamp; remaining TTL = `(taken_at + 24h) - now`; Vercel edge serves subsequent requests without hitting HikerAPI

---

## User Journey

```
Client's Existing Landing Page
    │  (CTA buttons link to our app)
    ▼
Plan Selection Page  ← we build this, styled to match client branding
    │
    ▼
Stripe Checkout (email required)
    │  Payment confirmed → Stripe webhook → create pending_registration record
    │                                     → send registration email via Resend (token link, 7-day expiry)
    ▼
Registration Page (/register?token=uuid, email pre-filled read-only)  ← we build this
    │  Token expired? → show "Resend link" option (no new charge)
    │
    ▼
Login Page  ← we build this
    │
    ▼
Dashboard
    │
    ├── Add Target (Instagram username)
    │     └── Validate: public? under follower limit? → save
    │
    ├── Manual Sync
    │     └── Fetch followers + following → diff → store events
    │
    ├── Activity Feed
    │     └── Who followed / unfollowed (last 90 days)
    │
    └── Story Viewer (plan-gated)
          └── Proxy media via HikerAPI → TTL-cached at Vercel edge → render in browser
```

> **Integration note:** We need the client to provide their landing page tech stack and share CTA button links/anchors so we can wire them to our `/plans`, `/register`, and `/login` routes.

---

## Deployment

### At Launch (Early Users)

| Service | Plan | Monthly Cost |
|---|---|---|
| Vercel | Hobby (free) | $0 |
| Supabase | Free tier | $0 |
| Inngest | Free tier (50k runs/mo) | $0 |
| Resend | Free tier (3k emails/mo) | $0 |
| Stripe | Pay as you go | ~2.9% + $0.30/txn |
| HikerAPI | Per request units | Variable |
| **Fixed total** | | **$0/month** |

### At 1000 Customers — Manual Sync

| Service | Plan | Monthly Cost | Why |
|---|---|---|---|
| Vercel | Hobby (free) | $0 | Bandwidth within limits |
| Supabase | Pro | $25 | followers_state ~3.2GB — free tier is 500MB |
| Inngest | Free tier | $0 | 18k runs/month — well within 50k free tier |
| Resend | Paid | $20 | ~4,300 emails/month — free tier is 3,000 |
| Stripe | Pay as you go | ~2.9% + $0.30/txn | — |
| HikerAPI | Per request units | Variable | — |
| **Fixed total** | | **$45/month** | |

### At 1000 Customers — Auto Sync (4-hour interval)

| Service | Plan | Monthly Cost | Why |
|---|---|---|---|
| Vercel | Hobby (free) | $0 | — |
| Supabase | Pro | $25 | — |
| Inngest | Paid | ~$52 | 360k runs/month (1000×2×6×30) |
| Resend | Paid | $20 | — |
| Stripe | Pay as you go | ~2.9% + $0.30/txn | — |
| HikerAPI | Per request units | Variable (higher) | More syncs = more API calls |
| **Fixed total** | | **~$97/month** | |

> Auto sync interval directly controls Inngest cost. 4-hour interval = $52/month. 30-minute interval = ~$556/month. Sync frequency must be priced into subscription plans.

---

## MVP Scope

**In scope:**
- Plan selection page + Stripe payment flow
- Login + registration pages (styled to match client's existing landing page)
- Plan enforcement (targets, follower count limits, sync limits, cooldowns)
- Manual sync with follower/following diff
- Activity feed showing follows/unfollows with usernames
- Story viewer with TTL-based edge caching (plan-gated)
- Weekly email summary
- Target status management (Active, Paused – Private, Paused – Out of Plan, etc.)

**Out of scope:**
- Landing page (client has it)
- Mobile app
- Real-time / automatic background syncing
- Push notifications (FCM/APNs)
- Facebook monitoring
- AI recommendations
- Admin/metrics dashboard
- Timeline visualizations

---

## Budget Breakdown

Implementation uses **BMAD v6** (AI-assisted development). Hours represent human developer time — AI handles boilerplate, scaffolding, and first-pass logic. Developer reviews, steers, tests, and handles integration complexity.

**BMAD v6 efficiency assumptions:**
- Boilerplate-heavy tasks (data models, CRUD, UI components): ~50% reduction
- Logic-heavy tasks (sync engine, diff algorithm, enforcement rules): ~30% reduction — these need careful human review regardless
- Testing/QA: ~25% reduction — human judgment still required

| Item | Without AI | With BMAD v6 | Notes | Cost |
|---|---|---|---|---|
| Project setup & service config | 4h | 2h | Next.js, Supabase, Stripe, Resend, subdomain config | $50 |
| Plan selection page + Stripe Checkout | 3h | 2h | Pricing cards → Stripe Checkout, styled to client brand | $50 |
| Auth flow: login + registration | 8h | 5h | Supabase Auth + post-payment registration gate | $125 |
| Plan management & enforcement | 4h | 2h | Plan rules, limits, enforcement middleware | $50 |
| Target management (add/remove/validate) | 8h | 4h | HikerAPI validation, status management | $100 |
| Sync engine — async + diff-only writes + Inngest | 14h | 10h | Inngest setup, sync function, Realtime, FOR UPDATE SKIP LOCKED, rate limit config, jitter | $250 |
| Story viewer (proxy + TTL cache + render) | 6h | 4h | HikerAPI proxy + Cache-Control TTL headers | $100 |
| Dashboard UI (activity feed, stats, targets) | 10h | 5h | Main app screens + Realtime sync status indicator | $125 |
| Weekly email (pg_cron + Resend template) | 4h | 2h | Conditional send logic + email template | $50 |
| Testing, QA, edge cases | 5h | 4h | Sync correctness, concurrency, limit enforcement, 1000-user load assumptions | $100 |
| **Total development** | **66h** | **40h** | | **$1,000** |
| AI model cost (1 month) | — | — | | $100 |
| **Total** | | | | **$1,100** |

**Budget remaining after delivery: $500**

This buffer covers:
- 1–2 client revision rounds
- 30-day post-launch bug fix window
- HikerAPI costs during testing/staging
- Supabase Pro upgrade when user count grows

---

## Risk Factors

| Risk | Mitigation |
|---|---|
| HikerAPI rate limits under concurrent load | Inngest built-in rate limiting (20 req/s global) + per-fetch jitter + Inngest auto-retry with exponential backoff on 429 |
| Supabase free tier storage exceeded | Upgrade to Pro ($25/mo) when approaching 500MB — triggered by user growth, not upfront |
| Concurrent sync race condition | FOR UPDATE SKIP LOCKED — DB-level lock, not application-level check |
| Edge Function timeout on large accounts | Plan-based follower cap bounds max job duration within 150s for all tiers up to 1M followers |
| Auto sync at scale | Inngest is already in the stack — auto sync is a feature toggle, not an architectural change. Cost must be reflected in subscription pricing. |
| Instagram account goes private mid-monitoring | Auto-pause detected on next sync with clear status message |
| Stripe webhook duplicate delivery | DB-level idempotency: UNIQUE(stripe_event_id) + INSERT ON CONFLICT DO NOTHING — application-level checks are raceable on serverless |
| Stripe webhook timeout causing retries | Return HTTP 200 immediately after DB insert; send registration email asynchronously (not inline) |
| Customer abandons registration or loses email | Registration email sent on payment with 7-day token link; expired link shows "Resend" option — no new charge |
| Webhook arrives after customer clicks register link | Register page polls for token record for up to 10 seconds ("Confirming payment…") before surfacing an error |
| Refund or chargeback — access not revoked | Handle charge.refunded and charge.dispute.created events; suspend subscription and revoke dashboard access |
| Registration email lands in spam or bounces | Resend bounce webhooks alert on delivery failure; DMARC + dedicated sending subdomain; post-checkout notice to "check your email" |
| Supabase getSession() auth bypass | All server/middleware code uses getUser() — makes live token verification call, cannot be bypassed with forged cookie |
| Service role key bundled into client | Admin Supabase client imported only in server-only files; never in components or client modules |
| Authenticated page cached by Vercel CDN | All /dashboard/* responses include Cache-Control: private, no-store |
| followers_state write load at scale | Diff-only writes (200 ops per sync) vs delete-all+insert-all (40k ops per sync) |
