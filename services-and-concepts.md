# Services & Concepts Guide

## Instagram Monitoring SaaS

This document explains every service in the stack and every technical concept discussed during planning — in plain language.

---

## Part 1: The Services

---

### 1. Next.js — The Application

**What it is:** A framework for building web applications using React. It handles both the pages users see and the server-side logic — in one codebase.

**What it does for us:**

Serves every page in the app:
- `/plans` — plan selection and pricing
- `/login` — user login
- `/register` — post-payment account creation
- `/dashboard` — main app (targets, activity feed, stats)
- `/dashboard/stories` — story viewer

Runs four server-side API routes:
- `/api/sync` — receives sync button click, validates it, hands the job to Inngest
- `/api/inngest` — the entry point Inngest calls to execute all background jobs
- `/api/stories` — fetches story media from HikerAPI and sends it to the browser
- `/api/webhook` — receives Stripe's payment confirmation and unlocks registration

**Why not a separate frontend and backend?**
The traditional approach is to build a React app (frontend) and a Node/Python server (backend) separately. That means two codebases, two deployments, and extra configuration to make them talk to each other. Next.js combines both. One codebase, one deployment.

---

### 2. Vercel — Hosting

**What it is:** The cloud platform that runs our Next.js app.

**What it does for us:**
- Automatically deploys the app every time we push code to git — no manual steps
- Serves the app on `app.domain.com` with free HTTPS
- Runs our API routes as serverless functions (they spin up when needed, shut down when idle — we never pay for idle time)
- **Edge caches story media** — when a user views stories for a target, Vercel caches the media at its global network of servers. The next user who views the same target's stories gets served from cache instantly, without hitting HikerAPI again

**The story cache works like this:**
Instagram stories expire 24 hours after they are posted. HikerAPI tells us when each story was posted (`taken_at` timestamp). We calculate how much time is left before it expires and cache it for exactly that long. A story posted 10 hours ago gets cached for 14 hours. A story posted 23 hours ago gets cached for 1 hour. When the story expires on Instagram, our cache expires automatically — no manual cleanup.

**In plain terms:** Vercel is the server that runs our app and the CDN that caches story content. We never manage or maintain a server.

---

### 3. Supabase — Database, Auth, and Real-time

Supabase is doing three separate jobs for us. Think of it as three services bundled into one platform.

---

#### 3a. PostgreSQL — The Database

**What it is:** A relational database. Stores data in tables with rows and columns, like a structured spreadsheet.

**What it stores for us:**

| Table | What it holds |
|---|---|
| `users` | Every registered account |
| `plans` | Plan tiers and their limits |
| `subscriptions` | Which user is on which plan |
| `targets` | Instagram accounts being monitored |
| `followers_state` | Current known follower list per target |
| `following_state` | Current known following list per target |
| `events` | Every follow/unfollow detected (kept 90 days) |
| `sync_runs` | One record per sync job — status, duration, cost |
| `daily_stats` | Aggregated follower counts per day (kept forever) |
| `email_log` | Record of every weekly email sent (prevents duplicates) |
| `pending_registrations` | Temporary record after payment, before account creation |
| `request_unit_usage` | HikerAPI cost tracking per user |

**In plain terms:** It is the single source of truth for everything in the application.

---

#### 3b. Auth — User Authentication

**What it is:** A ready-built authentication system.

**What it does for us:**
- Handles user login and registration
- Stores passwords securely (hashed — we never see the plain password)
- Issues session tokens when users log in
- Validates those tokens on every request

**The important part — Row Level Security (RLS):**
Supabase Auth integrates directly with the database. We write one rule per table:

```sql
-- Users can only see their own targets
CREATE POLICY "user_isolation" ON targets
  USING (user_id = auth.uid());
```

This rule lives in the database itself. It makes it physically impossible for one user to read another user's data — even if there is a bug in the application code. The database rejects the query before it runs.

**In plain terms:** We never store passwords ourselves. We never write login logic. Supabase handles it and the database enforces that users only see their own data.

---

#### 3c. Realtime — Live Updates

**What it is:** A system that pushes database changes to the browser instantly over a persistent connection.

**The problem it solves:**
When a user clicks Sync, the job takes 30–120 seconds to complete (Inngest runs it in the background). The user's browser needs to know when it finishes — without the user having to refresh the page.

**How it works:**
The browser opens a WebSocket connection to Supabase and says "tell me the moment this sync_run row changes." It waits silently. The moment Inngest writes `status = 'completed'` to that row, Supabase Realtime pushes that change to the browser instantly. The results appear on screen with no refresh and no repeated requests.

**In plain terms:** It is the live notification that tells the user "your sync is done, here are the results."

**Note — this is NOT SSE:** SSE (Server-Sent Events) is a similar technology but only goes one direction (server to browser). Supabase Realtime uses WebSockets, which is two-way. For our specific use case (just telling the browser the job is done), SSE would be sufficient — but we use Supabase Realtime because it is already in the stack and requires no extra setup.

---

#### 3d. pg_cron — Scheduled Database Task

**What it is:** A PostgreSQL extension that runs SQL on a schedule, like an alarm clock for the database.

**What it does for us:** One job only — deletes events older than 90 days. Runs every night at 2am UTC.

```sql
DELETE FROM events WHERE detected_at < now() - INTERVAL '90 days';
```

That's all. Everything else that used to run on a schedule (weekly email) has been moved to Inngest.

---

### 4. Inngest — Background Jobs

**What it is:** A managed platform for running background tasks — work that takes too long to happen inside a normal HTTP request.

**Why we need it:**
When a user clicks Sync, their browser sends a request to our server. Fetching hundreds of pages of follower data from HikerAPI takes 30–120 seconds. A normal HTTP request cannot stay open that long — the browser and server both give up after 10–60 seconds. Inngest solves this by taking the job off our hands and running it separately, with no time limit.

**What it does for us — three functions:**

---

**Function 1: `sync-target` (triggered by user clicking Sync)**

This is where the actual sync work happens:
1. Fetches the full follower list from HikerAPI, page by page, up to the plan's page cap
2. Fetches the full following list the same way
3. Computes who followed and who unfollowed since last sync (the diff)
4. Writes new events and updates the state tables in Supabase
5. Updates the sync_run record to `completed` — this triggers Supabase Realtime to notify the browser

Key things Inngest handles automatically for this function:
- **No timeout** — runs until the job is done, no matter how long
- **Rate limiting** — enforces a global limit of 20 HikerAPI requests per second across all concurrent sync jobs. If 50 users are syncing simultaneously, they do not collectively exceed this limit
- **Retries** — if HikerAPI returns an error or rate limit response (429), Inngest automatically retries with exponential backoff (waits 1 second, then 2, then 4, then 8)
- **Concurrency** — limits how many sync jobs run simultaneously (e.g. max 50 at a time), queuing the rest

---

**Function 2: `auto-sync-scheduler` (when auto sync is enabled per plan)**

Runs on a cron schedule (e.g. every 4 hours). Queries the database for all targets that are due for a sync. Sends one `sync/run` event per eligible target. Inngest queues all of them and processes them at the controlled concurrency limit — so thousands of targets don't all fire at the same moment.

---

**Function 3: `weekly-email`**

Runs every Monday at 9am UTC. Finds users who were active in the past 7 days and had follower/following changes. Builds a per-target summary and sends the email via Resend. Logs each sent email to prevent duplicates if the job runs twice.

**In plain terms:** Inngest is the engine room. All heavy work that can't happen in a user's HTTP request happens inside Inngest functions.

---

**Inngest cost:**
- Free tier: 50,000 job runs per month
- Manual sync at 1,000 users: ~18,000 runs/month — **free**
- Auto sync at 1,000 users (4-hour interval): ~360,000 runs/month — ~$52/month
- Auto sync interval directly controls cost. Shorter interval = more runs = higher cost

---

### 5. HikerAPI — Instagram Data

**What it is:** A third-party API service that fetches public Instagram data.

**What it does for us:**

| Action | When |
|---|---|
| Resolve username → user ID + follower count + public/private status | When user adds a target |
| Fetch full paginated followers list | During every sync |
| Fetch full paginated following list | During every sync |
| Fetch current public stories | When user opens story viewer |

**Why we use it instead of building our own scraper:**
Getting data from Instagram without their official API requires browser automation (Puppeteer or Playwright), residential proxy rotation, anti-bot handling, CAPTCHA solving, and constant maintenance every time Instagram updates its detection. Building this correctly takes 15–20 developer hours minimum, and then requires ongoing maintenance. HikerAPI has already built and maintained all of this. We call their API like any other REST service and receive clean structured data back.

**In plain terms:** It is our Instagram data source. Without it we would spend the entire budget building a scraper instead of building the product.

---

### 6. Stripe — Payments

**What it is:** A payment processing platform.

**What it does for us — two specific jobs:**

**Job 1: Checkout**
When a user selects a plan and clicks "Get Started", we redirect them to a Stripe-hosted Checkout page. Stripe collects their card details and processes the payment. We never touch card data — Stripe handles all of it including PCI compliance.

**Job 2: Webhook**
After a successful payment, Stripe sends a signed notification to our `/api/webhook` endpoint. We verify the signature to confirm it genuinely came from Stripe. We then create a `pending_registrations` record for that email address. This is the gate — a user can only create an account if a verified payment exists for their email. The record expires after 24 hours if they do not complete registration.

**The payment-first flow (unusual but client requirement):**
Most apps let you register first, then pay. This app requires payment before registration. Stripe makes this work cleanly — the Checkout session collects the email, payment succeeds, webhook fires, we store the email as "verified payer", user completes registration using that email.

**In plain terms:** Stripe is the cashier. It takes the money and tells us who paid. We check its records before letting anyone into the app.

---

### 7. Resend — Email Delivery

**What it is:** A transactional email service.

**What it does for us:** Delivers emails. Called by the Inngest `weekly-email` function every Monday.

For each eligible user, Resend sends an email showing:
- Per-target follower gains and losses for the past 7 days
- A link back to the dashboard

Email is only sent if the user logged in during the past week AND had activity. Users who did not log in or had no changes receive no email.

**Why Resend:**
- Free tier is 3,000 emails/month — covers launch
- Email templates are written in React (JSX) — fits naturally into our Next.js codebase
- Simple API

**In plain terms:** It is the delivery service for weekly summary emails. We tell it who to email and what to say. It handles getting it into their inbox.

---

## Part 2: Key Concepts Discussed

---

### SSR (Server-Side Rendering)

When a user visits a page, the server generates the complete HTML before sending it to the browser. The user sees content immediately without waiting for JavaScript to fetch data.

**In our app:** When a user opens `/dashboard`, Next.js fetches their targets and recent events on the server and sends a pre-built HTML page. Fast initial load.

This is about the **first thing the user sees when a page loads**.

---

### SSE (Server-Sent Events)

A technology where the server pushes updates to the browser over a persistent one-way HTTP connection. The browser cannot send data back over the same connection.

```
Browser opens connection → stays open
Server pushes messages to browser anytime
Browser cannot reply over the same connection
```

We discussed SSE in relation to Realtime. SSE would be technically sufficient for notifying the browser when a sync finishes (we only need one direction: server → browser). However we use Supabase Realtime instead because it is already in the stack with zero extra setup required.

---

### WebSocket

A persistent two-way connection between browser and server. Both sides can send messages at any time. This is the underlying technology Supabase Realtime uses.

```
Browser ◀──────────────────────▶ Server  (stays open, both can send)
```

**SSE vs WebSocket:**

| | SSE | WebSocket |
|---|---|---|
| Direction | Server → Browser only | Both ways |
| Protocol | HTTP | ws:// |
| Complexity | Simpler | More capable |
| Our use case | Sufficient | More than enough |

Supabase Realtime uses WebSocket. For our specific need (just notifying the browser when a sync is done) SSE would have worked too — but Realtime was already available in the stack.

---

### Async Sync Execution

"Async" means the work happens separately from the user's action — the user does not wait for it to finish.

**The problem:** Syncing an Instagram account takes 30–120 seconds. A browser request cannot stay open that long.

**The solution:**
```
User clicks Sync
    ↓
Server validates the request and creates a job record
Server tells Inngest "run this job"
Server immediately responds: "202 — job started" (takes < 1 second)
    ↓
Browser subscribes to Supabase Realtime and waits
    ↓
Inngest runs the sync in the background (30–120 seconds)
    ↓
Inngest writes "completed" to the job record in Supabase
    ↓
Supabase Realtime pushes the update to the browser
Browser shows results
```

The user sees a loading indicator, then results appear when the job finishes — without ever refreshing or waiting for an HTTP request.

---

### Concurrent Syncs

"Concurrent" means happening at the same time. Concurrent syncs = multiple users running a sync simultaneously.

**Why it matters:** All concurrent sync jobs share one HikerAPI API key. If too many jobs fire simultaneously they can collectively exceed HikerAPI's rate limit and start failing.

**How we handle it with Inngest:**
Inngest enforces a global rate limit of 20 HikerAPI requests per second across all running jobs. If 50 users sync at the same time, Inngest queues their requests and releases them at a controlled rate. None of the individual jobs know or care — Inngest handles the coordination.

---

### Scale: 1,000 Users Load Profile

When we say the architecture supports 1,000 users, here is what that actually means in numbers:

| Metric | Value | How we got there |
|---|---|---|
| Daily active users | 200 | 20% of 1,000 — typical SaaS benchmark |
| Avg targets per user | 2 | Mix of plan tiers |
| Daily syncs | 600 | 200 users × 3 syncs each |
| Busiest hour syncs | 60 | 10% of daily activity in one hour |
| Peak concurrent syncs | 10–15 | 60 syncs over 3,600 seconds, each lasting ~60s |
| followers_state rows | ~40 million | 1,000 × 2 targets × 20k avg followers |
| Database storage needed | ~3.2GB | 40M rows × 80 bytes |

The peak concurrent syncs figure (10–15) is what determines whether the infrastructure survives. At 10–15 simultaneous jobs, Inngest and Supabase handle it comfortably. The storage figure (3.2GB) is why Supabase needs to be upgraded from the free tier (500MB) to Pro ($25/month) as user count grows.

---

### Manual Sync vs Auto Sync

**Manual sync:** The user clicks a "Sync" button to fetch the latest data. Nothing happens automatically.

**Auto sync:** The system automatically syncs all monitored targets on a schedule (e.g. every 4 hours) without the user doing anything.

**Why we start with manual sync:**
Auto sync multiplies the number of jobs dramatically. At 1,000 users with 2 targets each and a 4-hour interval:

```
1,000 × 2 × 6 syncs/day × 30 days = 360,000 Inngest runs/month
```

That costs ~$52/month in Inngest alone and significantly more in HikerAPI request units. These costs must be built into subscription pricing before enabling the feature.

Manual sync at 1,000 users produces only ~18,000 runs/month — within Inngest's free tier.

**Auto sync is not an architectural change.** With Inngest already in the stack, enabling auto sync is adding one scheduled function in code and adjusting pricing. The infrastructure is ready for it.

---

### Diff-Only Writes

Every sync needs to update the `followers_state` table to reflect the current follower list. There are two ways to do this:

**Old approach — delete all, insert all:**
```
Delete all 20,000 rows for this target
Insert all 20,000 current followers
= 40,000 database operations per sync
```

**Our approach — write only what changed:**
```
Compare current list against stored list
Find who is new → insert only those rows (e.g. 50 new followers)
Find who left → delete only those rows (e.g. 20 unfollows)
= ~70 database operations per sync
```

At 1,000 users with 600 syncs per day:
- Delete all + insert all: **24 million** database operations per day
- Diff-only: **~42,000** database operations per day

This is a **570× reduction** in database write load. At scale this is the difference between a database that is fast and one that struggles.

---

### Plan-Tier → Page Cap → Job Duration

Each plan tier limits the maximum follower count for monitored accounts. This limit directly controls how long a sync job can take:

| Plan | Max Followers | Pages Fetched | Approx Duration |
|---|---|---|---|
| Basic | 50,000 | 25 pages | ~15 seconds |
| Pro | 500,000 | 100 pages | ~70 seconds |
| Enterprise | 1,000,000 | 200 pages | ~140 seconds |

MVP maximum: **1 million followers**. Accounts above this are rejected when added. Beyond 1 million, a different architecture (dedicated long-running worker) is needed — a post-MVP decision.

---

### Monthly Infrastructure Cost Summary

| | At Launch | 1,000 Users (manual sync) | 1,000 Users (auto sync, 4h) |
|---|---|---|---|
| Vercel | $0 | $0 | $0 |
| Supabase | $0 | $25 | $25 |
| Inngest | $0 | $0 | $52 |
| Resend | $0 | $20 | $20 |
| **Fixed total** | **$0** | **$45** | **$97** |

HikerAPI and Stripe costs are variable (per usage and per transaction) and should be factored into subscription pricing.
