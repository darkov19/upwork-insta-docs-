# Development Plan

## Instagram Monitoring SaaS

---

## Context & Decision

The original proposal listed Login, Registration, and Stripe Checkout as **client prerequisites** — work the client would handle before we start. In practice, the client is unlikely to build these themselves.

**Decision:** Start development now without waiting. Build the entire product against a dummy user using an auth stub. Auth will be plugged in as an isolated swap at the end — either when the client delivers it, or as a paid add-on when they confirm they cannot.

This eliminates the blocking dependency and keeps the project moving. Everything is built user-scoped from day one, so dropping in real auth later requires changing one file.

---

## Auth Stub Approach

Instead of a real login flow, we create a thin middleware stub:

```
middleware.ts (stub)
  → reads DEV_USER_ID from environment variables
  → injects it as the authenticated user on every /dashboard/* request
  → all API routes and server components call getUser() as normal
  → they receive the test user back
  → zero app-level changes needed when real auth replaces this stub
```

**Dummy user setup (seeded once in Supabase):**

- One real Supabase Auth user created via the Supabase dashboard (`dev@test.com`)
- One `subscriptions` row seeded for that user pointing to a plan
- `DEV_USER_ID` env var set to that user's UUID

When real auth is ready: delete the stub middleware, add Supabase session check. Done.

---

## Phase 1 — Foundation

**Goal:** Everything running locally. Database ready. Dummy user in place. All services wired.

| Task                        | Notes                                                                        |
| --------------------------- | ---------------------------------------------------------------------------- |
| Next.js project scaffolding | App Router, TypeScript, Tailwind                                             |
| Supabase project setup      | Dev project; create prod project before go-live                              |
| Inngest local dev server    | `inngest dev` running alongside Next.js                                      |
| Stripe CLI setup            | Webhook forwarding for local testing                                         |
| Resend account wired        | Dev API key; prod domain configured before go-live                           |
| Database schema             | All 10 tables with indexes and RLS policies                                  |
| Seed data                   | `plans` table with placeholder tier values (client to confirm exact numbers) |
| Dummy user + subscription   | `dev@test.com` in Supabase Auth + seeded `subscriptions` row                 |
| Auth stub                   | Middleware returning hardcoded test user                                     |
| Environment variables       | All service keys wired in `.env.local`                                       |

---

## Phase 2 — Core Engine

**Goal:** Sync working end-to-end against the dummy user.

| Task                    | Notes                                                                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| HikerAPI discovery      | Read docs, make test calls, understand pagination cursor, request unit tracking, edge cases (private accounts, 0 followers, last page detection) |
| Target management       | Add by username, HikerAPI validation (public? under follower limit?), all 5 status states, remove target                                         |
| Sync engine             | Full async flow: `/api/sync` → Inngest → followers/following diff → events → `daily_stats` → Realtime push                                       |
| Plan enforcement        | Cooldown check, daily limit check, concurrent sync guard (`FOR UPDATE SKIP LOCKED`), page cap                                                    |
| Inngest function config | Rate limit (20 req/s), retries (3, exponential backoff), concurrency (50 max)                                                                    |
| Supabase Realtime       | Frontend subscribes to `sync_runs` row — displays live sync status, completion notification                                                      |

---

## Phase 3 — Dashboard + Remaining Features

**Goal:** Full working product. Everything except auth and Stripe Checkout.

| Task                                         | Notes                                                                                                                               |
| -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Dashboard UI                                 | Target list (5 status states, cooldown timer), activity feed (events, pagination, empty state), stats display, plan usage indicator |
| Story viewer                                 | `/api/stories/[targetId]` — HikerAPI proxy + dynamic TTL Cache-Control headers; plan gate                                           |
| Weekly email                                 | Inngest cron (Monday 9am UTC), Resend template, `email_log` idempotency, conditional send logic                                     |
| Stripe subscription webhooks                 | `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`, downgrade auto-pause logic              |
| `charge.refunded` + `charge.dispute.created` | Suspend subscription, revoke dashboard access                                                                                       |
| Client branding                              | Apply client logo, colours, and fonts across all pages                                                                              |
| Deployment                                   | Vercel prod, Supabase prod project, Stripe prod webhook endpoint, `app.domain.com` subdomain DNS                                    |

---

## Phase 4 — Auth (Separate Milestone)

**Trigger:** Client confirms they cannot build it, or requests it as an add-on.

**Quote: $275–$300 as a separate Upwork milestone.**

Justification is clean: auth was explicitly listed as a client prerequisite in the proposal (Section 5). The app is complete. This is the one remaining piece.

| Task                          | Notes                                                                                                           |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Remove auth stub              | Delete stub middleware, add real Supabase session check                                                         |
| Login page                    | `/login` — email/password form, `signInWithPassword`, error states, redirect to `/dashboard`                    |
| Register page                 | `/register?token=uuid` — email pre-filled (read-only), password form, token validation, atomic account creation |
| Stripe webhook → registration | `checkout.session.completed` → `pending_registrations` row + async registration email via Resend                |
| Registration email            | Token link (7-day expiry), tracking disabled, bounce webhook handling                                           |
| Forgot password               | Supabase built-in reset flow wired to a UI page                                                                 |
| `/api/resend-registration`    | Recovery endpoint — re-issues token for paid users whose link expired                                           |

---

## Client Communication

### Message to send now (before starting):

> "We're starting development this week. As outlined in the proposal (Section 5), the plan selection page, Stripe Checkout, and login/registration pages are on your side to build. We'll deliver everything else — sync engine, dashboard, story viewer, plan enforcement, and weekly emails. We'll keep you posted on progress and will need the plan tier values (follower limits, cooldowns, daily sync limits) confirmed before we wire up enforcement logic."

This sets expectations in writing, reminds them of their obligation, and starts the clock on them realising they cannot deliver it.

### When they come back unable to build auth:

> "As outlined in the proposal (Section 5), login and registration were listed as client prerequisites. We've built and delivered everything else — the app is ready to launch. The only remaining piece is the auth flow. We can handle this as an add-on milestone for $[275–300]. This covers the registration flow, login page, forgot password, and the Stripe webhook integration that creates user accounts after payment."

---

## Open Items Still Needed From Client

These must be confirmed before enforcement logic is finalised:

| #   | Item                                   | Blocks                                               |
| --- | -------------------------------------- | ---------------------------------------------------- |
| 1   | Follower count limit per plan tier     | Target add validation, `plans` table seed            |
| 2   | Cooldown duration per plan tier        | Sync cooldown check                                  |
| 3   | Daily sync limit per plan tier         | Daily limit check                                    |
| 4   | Page cap per plan tier                 | HikerAPI cost per sync, sync engine                  |
| 5   | Timezone for weekly emails             | Currently UTC — flag if user-local timezone required |
| 6   | Branding assets (logo, colours, fonts) | Client branding integration                          |

Placeholder values will be seeded during Phase 1 so development is not blocked. Client confirms exact values before go-live.

---

## Service Accounts Needed (Onboarding Checklist)

| Service                                    | Who creates | When needed                          |
| ------------------------------------------ | ----------- | ------------------------------------ |
| Supabase (dev + prod)                      | Developer   | Phase 1                              |
| Inngest (dev + prod)                       | Developer   | Phase 1                              |
| Vercel                                     | Developer   | Phase 3 (deployment)                 |
| HikerAPI                                   | Client      | Phase 2 (before sync engine testing) |
| Stripe (with products + prices configured) | Client      | Phase 3 (webhook testing)            |
| Resend (with verified sending domain)      | Client      | Phase 3 (email testing)              |
| `app.domain.com` subdomain DNS → Vercel    | Client      | Phase 3 (go-live)                    |
