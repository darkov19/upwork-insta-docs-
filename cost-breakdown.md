# Cost Breakdown & Quote Analysis

## Instagram Monitoring SaaS — Upwork Engagement

---

## Internal Cost Baseline

### Developer Rate & Tooling

| Item                      | Value                         |
| ------------------------- | ----------------------------- |
| Developer hourly rate     | $25/hr                        |
| AI tooling                | BMAD v6 (Claude)              |
| Claude API cost (1 month) | $100                          |
| Platform                  | Upwork (fixed-price contract) |

### BMAD v6 Efficiency Assumptions

| Task type                                              | Hour reduction |
| ------------------------------------------------------ | -------------- |
| Boilerplate-heavy (data models, CRUD, UI components)   | ~50%           |
| Logic-heavy (sync engine, diff algorithm, enforcement) | ~30%           |
| Testing & QA                                           | ~20%           |
| Discovery, deployment, integration                     | ~15–20%        |

---

## Hour Estimate Audit

The original estimate was **40 hours**. The audit below identified 4 missing line items and significant underestimates on UI and QA.

| Item                                  | Without AI | With BMAD v6 | Original BMAD | Change   | Notes                                                                               |
| ------------------------------------- | ---------- | ------------ | ------------- | -------- | ----------------------------------------------------------------------------------- |
| Project setup & service config        | 5h         | 3h           | 2h            | +1h      | Inngest local dev server, Stripe CLI, all 5 services wired together                 |
| **Database schema + RLS policies**    | 5h         | 3h           | _(missing)_   | +3h      | 10 tables, indexes, RLS policies per table — separate from setup                    |
| Plan selection + Stripe Checkout      | 6h         | 4h           | 2h            | +2h      | Stripe webhook handler, idempotency, pending_registration creation                  |
| Auth flow: login + registration       | 10h        | 7h           | 5h            | +2h      | Supabase SSR session pattern, post-payment registration edge cases                  |
| Plan management + Stripe sub webhooks | 7h         | 5h           | 2h            | +3h      | subscription.updated, subscription.deleted, invoice.payment_failed, downgrade logic |
| Target management                     | 10h        | 6h           | 4h            | +2h      | 5 status states, cooldown timer UI, HikerAPI validation edge cases                  |
| Sync engine + Inngest                 | 18h        | 12h          | 10h           | +2h      | Full async loop: /api/sync → Inngest → Realtime → UI; integration testing           |
| Story viewer                          | 6h         | 4h           | 4h            | 0h       | Accurately estimated — TTL math and proxy pattern are well-defined                  |
| Dashboard UI                          | 12h        | 9h           | 5h            | +4h      | Activity feed + pagination, Realtime sync status, plan usage display, all states    |
| Weekly email                          | 5h         | 3h           | 2h            | +1h      | React Email template, Inngest cron, email_log idempotency                           |
| Testing & QA                          | 12h        | 8h           | 4h            | +4h      | 10 test scenarios — each requires setup, execution, verification                    |
| **HikerAPI discovery**                | 3h         | 2h           | _(missing)_   | +2h      | Docs, test calls, response format, pagination cursor, request unit tracking         |
| **Client branding integration**       | 5h         | 3h           | _(missing)_   | +3h      | Apply logo, colours, fonts across all pages; client approval round                  |
| **Deployment + go-live**              | 4h         | 3h           | _(missing)_   | +3h      | Prod env vars, Stripe prod webhook, Supabase prod project, subdomain DNS            |
| **Total**                             | **108h**   | **72h**      | **40h**       | **+32h** |                                                                                     |

---

## What Each Underestimated Item Actually Entails

### Database schema + RLS (was missing)

Row Level Security policies are error-prone — a wrong policy silently leaks data or silently blocks queries. 10 tables each need at minimum a `SELECT` policy scoped to `auth.uid()`. Some tables (e.g. `targets`, `events`) need additional policies for `INSERT`, `UPDATE`, `DELETE`. Testing each policy requires running queries as both the correct user and an unauthorized user. This is not 30 minutes inside project setup.

### Plan management + Stripe subscription webhooks (was 2h)

The original estimate only considered enforcing plan limits — it did not account for the Stripe subscription lifecycle:

- `customer.subscription.updated` → update plan tier, re-evaluate target eligibility
- `customer.subscription.deleted` → cancel access, handle data retention
- `invoice.payment_failed` → mark `past_due`, gate dashboard access
- Downgrade logic: user drops from Pro (5 targets) to Basic (2 targets) — auto-pause 3 excess targets with correct status

### Dashboard UI (was 5h)

5 hours for a full application's UI was the most underestimated line:

| Screen / component                                          | Time   |
| ----------------------------------------------------------- | ------ |
| Layout + navigation + header                                | 1h     |
| Target list (5 status states, cooldown timer, add/remove)   | 2h     |
| Activity feed (events, pagination, timestamps, empty state) | 2.5h   |
| Realtime sync status (spinner, completion notification)     | 1h     |
| Stats display + plan usage + upgrade prompt                 | 1.5h   |
| **Total**                                                   | **9h** |

### Testing & QA (was 4h)

4 hours cannot cover 10 distinct test scenarios. Each requires environment setup, execution, and verification:

| Scenario                                              | Time    |
| ----------------------------------------------------- | ------- |
| Sync correctness — correct diff, correct events       | 1h      |
| Concurrent sync — FOR UPDATE SKIP LOCKED works        | 45min   |
| Cooldown enforcement                                  | 30min   |
| Daily sync limit enforcement                          | 30min   |
| Plan limits — target cap, story viewer gate           | 45min   |
| Stripe webhook replay via stripe-cli                  | 1h      |
| Registration edge cases — expired token, already used | 45min   |
| Partial sync — page cap hit, completed_partial status | 45min   |
| Private account detection during sync                 | 30min   |
| End-to-end full journey smoke test                    | 1h      |
| **Total**                                             | **~8h** |

### HikerAPI discovery (was missing)

Before writing sync code, the developer must:

- Read HikerAPI docs for followers, following, profile, and stories endpoints
- Make exploratory test calls and understand the response format
- Understand the pagination cursor pattern (how to detect last page, cursor expiry)
- Understand how `request_units_consumed` is returned per response
- Test edge cases in the API: private accounts, accounts with 0 followers, cursor at last page

Skipping this causes surprises mid-implementation. 2 hours before writing the sync engine is not optional.

---

## Cost Summary

|                          | Original estimate | Revised estimate |
| ------------------------ | ----------------- | ---------------- |
| Dev hours (with BMAD v6) | 40h               | 72h              |
| Dev cost ($25/hr)        | $1,000            | $1,800           |
| Claude API (1 month)     | $100              | $100             |
| **Total internal cost**  | **$1,100**        | **$1,900**       |

---

## Upwork Fee Impact

Upwork charges the freelancer a service fee on earnings from each client:

| Earnings with this client (lifetime) | Upwork fee | You keep   |
| ------------------------------------ | ---------- | ---------- |
| First $500                           | 20%        | 80% ($400) |
| $500.01 – $10,000                    | 10%        | 90%        |
| Above $10,000                        | 5%         | 95%        |

For a new client relationship, the 20%/10% split applies.

### What different quote amounts net you (new client)

| Client pays | You receive (after Upwork) | After Claude ($100) | Dev hours covered | Verdict                                     |
| ----------- | -------------------------- | ------------------- | ----------------- | ------------------------------------------- |
| $1,000      | $850                       | $750                | 30h               | 42h short — working at ~$18.75/hr effective |
| $1,500      | $1,300                     | $1,200              | 48h               | 24h short of revised scope                  |
| $2,000      | $1,750                     | $1,650              | 66h               | Close to break-even on revised scope        |
| **$2,200**  | **$1,930**                 | **$1,830**          | **73h**           | **Break-even on 72h revised scope**         |
| $2,500      | $2,200                     | $2,100              | 84h               | 12h revision buffer                         |

### To net $1,900 (break-even) after Upwork fees

```
First $500 billed → you keep $400
Remaining $1,500 needed ÷ 0.90 = $1,667 to bill
─────────────────────────────────────────────────
Minimum quote to break even: $500 + $1,667 = $2,167
```

---

## Quote Options

### Option A — $2,200 (minimum viable)

- Covers full 72h scope + Claude
- No revision buffer
- Any scope creep comes out of your pocket
- Risk: medium (one unexpected integration issue eats your margin)

### Option B — $2,500 (recommended)

- Covers 72h scope + Claude + ~12h buffer
- Buffer absorbs: 1 revision round, one integration surprise, minor client change requests
- Risk: low
- Position to client: reflects BMAD v6 savings from 108h without AI

### Option C — $3,000 (comfortable)

- Covers 72h scope + Claude + 30h buffer
- Comfortable for a first engagement where requirements may shift
- Risk: very low
- May feel high to client — requires stronger justification

---

## Recommended Quote: $2,500 fixed price

Use this framing on Upwork:

> _"Without AI-assisted development (BMAD v6), this project is a 108-hour engagement. We use AI tooling to deliver the same output in 72 hours. The Claude API cost is absorbed into the quote. The $2,500 fixed price reflects the reduced hours — you are not paying for the time the AI saves."_

---

## Infrastructure Costs (Client Pays Directly — Not in Quote)

These are the client's own SaaS accounts. Make this explicit in the Upwork proposal to avoid the client expecting you to cover HikerAPI bills.

### At launch (early users)

| Service         | Plan                | Monthly cost      |
| --------------- | ------------------- | ----------------- |
| Vercel          | Hobby               | $0                |
| Supabase        | Free                | $0                |
| Inngest         | Free (50k runs/mo)  | $0                |
| Resend          | Free (3k emails/mo) | $0                |
| Stripe          | Pay-as-you-go       | ~2.9% + $0.30/txn |
| HikerAPI        | Per request unit    | Variable          |
| **Fixed total** |                     | **$0/month**      |

### At 1,000 customers (manual sync)

| Service         | Plan             | Monthly cost      |
| --------------- | ---------------- | ----------------- |
| Vercel          | Hobby            | $0                |
| Supabase        | Pro              | $25               |
| Inngest         | Free             | $0                |
| Resend          | Paid             | $20               |
| Stripe          | Pay-as-you-go    | ~2.9% + $0.30/txn |
| HikerAPI        | Per request unit | Variable          |
| **Fixed total** |                  | **$45/month**     |

---

## If Client's Hard Ceiling Is $1,000

A $1,000 client budget is not viable for this scope.

After Upwork fees: you receive $850. After Claude: $750 net. At $25/hr: 30 hours of dev time. The revised scope is 72 hours. **You would be funding 42 hours yourself.**

Two options:

**Option A — Reduce scope to fit ~30 hours**

Features to defer post-MVP:

| Cut                                                  | Hours saved              |
| ---------------------------------------------------- | ------------------------ |
| Story viewer                                         | -4h                      |
| Weekly email                                         | -3h                      |
| Client branding (deliver unstyled, client styles it) | -3h                      |
| Reduce QA from 8h → 4h                               | -4h                      |
| **Total saved**                                      | **-14h → 58h remaining** |

58h still far exceeds the 30h budget. Scope reduction alone cannot bridge a 42-hour gap. This option does not work cleanly.

**Option B — Decline and explain**

> _"The MVP scope requires 72 development hours with AI tooling. Our minimum viable quote is $2,200. We can discuss deferring features like the story viewer and weekly email to reduce scope, but the core sync engine, auth, and Stripe integration alone account for ~45 hours and cannot be reduced further without compromising correctness."_

---

## If Client Offers $1,500–$1,600

### The Math

|                               | $1,500 quote   | $1,600 quote     |
| ----------------------------- | -------------- | ---------------- |
| Upwork fee (first $500 @ 20%) | -$100          | -$100            |
| Upwork fee (remainder @ 10%)  | -$100          | -$110            |
| You receive                   | $1,300         | $1,390           |
| Claude API                    | -$100          | -$100            |
| **Net in your pocket**        | **$1,200**     | **$1,290**       |
| At $25/hr: hours covered      | **48h**        | **51.6h**        |
| Revised scope needs           | 72h            | 72h              |
| **Hours you fund yourself**   | **24h ($600)** | **20.4h ($510)** |
| Effective hourly rate         | **$16.67/hr**  | **$17.92/hr**    |

You would be working roughly one-third of the project for free.

---

### Option 1 — Accept It, Cut Scope to Fit 48–52 Hours

To fit within the hours the budget covers, these features must be cut from the contract:

| Cut                      | Hours saved        | Impact                                  |
| ------------------------ | ------------------ | --------------------------------------- |
| Story viewer             | -4h                | Delivered in Phase 2                    |
| Weekly email             | -3h                | Delivered in Phase 2                    |
| Client branding          | -3h                | Basic functional styling only           |
| Activity feed pagination | -1.5h              | Show last 50 events, no load more       |
| Stats display            | -1h                | Remove from dashboard                   |
| Plan usage display       | -1h                | Remove from dashboard                   |
| QA: 8h → 4h              | -4h                | Smoke test only, no edge case scenarios |
| **Total saved**          | **-17.5h → 54.5h** |                                         |

Even after all these cuts you're at **54.5h** — still 3–6 hours over a $1,500–1,600 budget. The client gets a working product but not the full MVP they expect: no story viewer, no weekly email, no branding, minimal dashboard, light QA.

---

### Option 2 — Accept It, Work the Full 72 Hours

Take the project at $1,500–$1,600, deliver the full 72h scope, accept the lower effective rate.

**When this makes sense:**

- This is your first Upwork client — a 5-star review is worth real money in future contracts
- You're learning the stack (Inngest, Supabase Realtime, HikerAPI) — effectively paid training
- You want the portfolio piece

**When it doesn't:**

- You have other paid work at $25/hr — the 20–24 unpaid hours have an opportunity cost
- The client has revision expectations — any revision round pushes you further negative

---

### Option 3 — Phase It (Recommended)

Accept $1,500–$1,600 but split delivery into two contracts.

**Phase 1 — $1,600 (this contract):**

| Item                                                           | Hours   |
| -------------------------------------------------------------- | ------- |
| Project setup + DB schema                                      | 6h      |
| Auth flow: login + registration                                | 7h      |
| Plan selection + Stripe Checkout                               | 4h      |
| Plan management + Stripe webhooks                              | 5h      |
| Target management                                              | 6h      |
| Sync engine + Inngest + Realtime                               | 12h     |
| Dashboard UI (core only — targets, activity feed, sync button) | 6h      |
| Deployment + go-live                                           | 3h      |
| QA (core flows only)                                           | 4h      |
| **Phase 1 total**                                              | **53h** |

Tight at $1,600 (51.6h covered) — you absorb ~1.5h. Acceptable.

**Phase 2 — Separate quote ~$550–$600 (after launch):**

| Item                                                   | Hours   |
| ------------------------------------------------------ | ------- |
| Story viewer (proxy + TTL cache + UI)                  | 4h      |
| Weekly email (Inngest cron + Resend template)          | 3h      |
| Client branding integration                            | 3h      |
| Dashboard enhancements (stats, plan usage, pagination) | 3h      |
| Full QA pass                                           | 4h      |
| **Phase 2 total**                                      | **17h** |

**Total client pays: $2,150–$2,200** — same destination as the recommended quote, split into two contracts. Easier for a client with a tight initial budget to say yes to Phase 1 now and Phase 2 after they see it working.

**What to say to the client:**

> _"We can start with the core MVP at $1,600 — this covers the sync engine, Stripe payment flow, auth, target management, and dashboard. Story viewer, weekly email, and branding polish would be a Phase 2 at around $600 once the core is live and you've validated it with early users. Total comes to $2,200, same as the full quote, but split so you can see working software before committing the remainder."_

---

### Comparison

| Approach     | Quote                       | You work      | Effective rate | Delivers                                        |
| ------------ | --------------------------- | ------------- | -------------- | ----------------------------------------------- |
| Cut scope    | $1,500                      | 48h           | $25/hr         | Stripped MVP — no story viewer, email, branding |
| Accept loss  | $1,500                      | 72h           | $16.67/hr      | Full MVP — 24h unpaid                           |
| Accept loss  | $1,600                      | 72h           | $17.92/hr      | Full MVP — 20h unpaid                           |
| **Phase it** | **$1,600 now + $600 later** | **72h total** | **~$25/hr**    | **Full MVP — split across 2 contracts**         |

The phased approach is the only path that delivers the full product, protects your hourly rate, and works within the client's stated budget ceiling.

---

## Summary

|                                        | Value                                         |
| -------------------------------------- | --------------------------------------------- |
| Revised dev hours (BMAD v6)            | 72h                                           |
| Revised internal cost (dev + Claude)   | $1,900                                        |
| Minimum Upwork quote to break even     | $2,200                                        |
| Recommended Upwork quote (with buffer) | **$2,500**                                    |
| If client offers $1,500–$1,600         | Phase the work — $1,600 now + $600 Phase 2    |
| Original estimate (underestimated)     | 40h / $1,100                                  |
| Hours missing from original estimate   | +32h (4 missing items + UI/QA underestimates) |
