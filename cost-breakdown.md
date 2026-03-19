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

| Item                                               | Without AI | With BMAD v6 | Original BMAD | Change   | Notes                                                                               |
| -------------------------------------------------- | ---------- | ------------ | ------------- | -------- | ----------------------------------------------------------------------------------- |
| Project setup & service config                     | 5h         | 3h           | 2h            | +1h      | Inngest local dev server, Stripe CLI, all 5 services wired together                 |
| **Database schema + RLS policies**                 | 5h         | 3h           | _(missing)_   | +3h      | 10 tables, indexes, RLS policies per table — separate from setup                    |
| ~~Plan selection + Stripe Checkout~~               | ~~6h~~     | ~~4h~~       | ~~2h~~        | —        | **Excluded from this contract — client handles**                                    |
| ~~Auth flow: login + registration~~                | ~~10h~~    | ~~7h~~       | ~~5h~~        | —        | **Excluded from this contract — client handles**                                    |
| Plan management + Stripe sub webhooks              | 7h         | 5h           | 2h            | +3h      | subscription.updated, subscription.deleted, invoice.payment_failed, downgrade logic |
| Target management                                  | 10h        | 6h           | 4h            | +2h      | 5 status states, cooldown timer UI, HikerAPI validation edge cases                  |
| Sync engine + Inngest                              | 18h        | 12h          | 10h           | +2h      | Full async loop: /api/sync → Inngest → Realtime → UI; integration testing           |
| Story viewer                                       | 6h         | 4h           | 4h            | 0h       | Accurately estimated — TTL math and proxy pattern are well-defined                  |
| Dashboard UI                                       | 12h        | 9h           | 5h            | +4h      | Activity feed + pagination, Realtime sync status, plan usage display, all states    |
| Weekly email                                       | 5h         | 3h           | 2h            | +1h      | React Email template, Inngest cron, email_log idempotency                           |
| Testing & QA                                       | 10h        | 6h           | 4h            |          | Auth + payment flows excluded from QA scope — 8 scenarios remain                   |
| **HikerAPI discovery**                             | 3h         | 2h           | _(missing)_   | +2h      | Docs, test calls, response format, pagination cursor, request unit tracking         |
| **Client branding integration**                    | 3h         | 2h           | _(missing)_   |          | Fewer pages (no auth/payment pages) — reduced from 3h                               |
| **Deployment + go-live**                           | 4h         | 3h           | _(missing)_   | +3h      | Prod env vars, Stripe prod webhook, Supabase prod project, subdomain DNS            |
| **Total (in-scope)**                               | **83h**    | **58h**      | —             |          | Excludes 11h for Stripe Checkout + Auth (client-handled)                            |

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

|                               | Original estimate | Full revised | In-scope (excl. Stripe + Auth) |
| ----------------------------- | ----------------- | ------------ | ------------------------------ |
| Dev hours (with BMAD v6)      | 40h               | 72h          | **58h**                        |
| Dev cost ($25/hr)             | $1,000            | $1,800       | **$1,450**                     |
| Claude API (1 month)          | $100              | $100         | $100                           |
| **Total internal cost**       | **$1,100**        | **$1,900**   | **$1,550**                     |

---

## Upwork Fee Impact

> **Updated for 2025 fee structure.** As of May 1, 2025, Upwork replaced the old tiered 20%/10%/5% system with a **variable flat rate of 0–15%** per contract. The rate is determined by Upwork based on skill demand, market saturation, and project complexity — it is shown to you before submitting a proposal and remains fixed for the duration of that contract.
>
> The old tiered calculation in this document is no longer applicable.

### New Fee Structure (Effective May 1, 2025)

| Fee range | Who gets it                                   |
| --------- | --------------------------------------------- |
| 0%        | High-demand / undersupplied skill categories  |
| ~10–13%   | Typical rate for most development contracts   |
| 15%       | Oversaturated categories                      |

The exact rate for this contract was visible at proposal submission time. If the rate was not recorded, use the 10–13% range for planning.

### Actual deductions for this contract

| Deduction                  | Rate  | Amount on $1,600 |
| -------------------------- | ----- | ---------------- |
| Upwork service fee         | 10.0% | $160.00          |
| Indian TDS (tax deducted)  | 0.5%  | $8.00            |
| **Total deducted**         | **10.5%** | **$168.00**  |
| **You receive**            |       | **$1,432.00**    |

### To net $1,550 (break-even) after all deductions

```
Required net ÷ (1 − 0.105) = $1,550 ÷ 0.895 = $1,732 → quote $1,750
```

---

## Quote

### Recommended Quote: $1,600 fixed price

- Covers 58h in-scope work with ~6h absorbed (~$160 below break-even)
- Gap is acceptable: reduced QA scope and fewer pages means realistic hours are closer to 54–56h
- Minimum break-even quote is $1,750 — going to $1,600 is a deliberate decision for client fit
- Risk: low (gap is small, scope is well-defined)

> **Note:** Plan Selection + Stripe Checkout and Auth flow are excluded from this contract. If the client later needs these built, quote separately (~$550 based on 11h at $25/hr with buffer).

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

## Quote Decision — $1,600 Fixed Price

| Item                          | Amount           |
| ----------------------------- | ---------------- |
| Quote                         | $1,600.00        |
| Upwork service fee (10%)      | -$160.00         |
| Indian TDS (0.5%)             | -$8.00           |
| **You receive**               | **$1,432.00**    |
| Claude API                    | -$100.00         |
| **Net in your pocket**        | **$1,332.00**    |
| At $25/hr: hours covered      | 53.3h            |
| In-scope hours                | 58h              |
| **Hours absorbed**            | **4.7h (~$118)** |
| Effective hourly rate         | $22.96/hr        |

The gap is ~4.7 hours — less than one day. Scope is well-defined and client prerequisites (auth, payment) are already handled. Accepted.

---

## Summary

|                                        | Value                                                          |
| -------------------------------------- | -------------------------------------------------------------- |
| In-scope dev hours (BMAD v6)           | 58h (72h full scope − 11h client-handled − 3h QA/branding trim)|
| In-scope internal cost (dev + Claude)  | $1,550                                                         |
| Minimum Upwork quote to break even     | $1,750                                                         |
| Recommended Upwork quote               | **$1,600**                                                     |
| Excluded from this contract            | Plan Selection + Stripe Checkout (4h), Auth flow (7h) = 11h   |
| If excluded modules needed later       | ~$550 separate quote (11h + buffer)                            |
| Original estimate (underestimated)     | 40h / $1,100                                                   |
