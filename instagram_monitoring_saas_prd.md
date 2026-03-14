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
4. Registration (Account creation)
5. Dashboard Access

### Important Notes

- Payment must be associated with email.
- If user abandons registration after payment, the payment session must expire after a configurable duration (e.g., 24 hours).

---

## 4. Subscription & Plans

Each plan defines:

- `maxTargets`
- `maxFollowerCountAllowed`
- `storyViewerEnabled`
- `dailySyncLimitPerUser`
- `cooldownMinutesBetweenSyncs`

### Enforcement Rules

- Target cannot be added if `follower_count` exceeds plan threshold.
- If target later exceeds threshold → status = **"Paused – Out of Plan"**.
- Private accounts are rejected on add.
- If account becomes private → monitoring paused.

---

## 5. Sync Model

### Manual Sync Only

There is **NO automatic background syncing**.

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

Available only if plan allows.

#### Capabilities

- Fetch current public stories.
- Render media inside the app.
- Do not store media permanently.
- Media must be proxied through backend.
- No long-term caching.

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

- Hard page cap per sync (configurable).
- Daily sync cap.
- Cooldown enforcement.
- Stop sync if request unit usage exceeds threshold.

### Metrics Tracked

- Cost per user
- Average syncs per user
- Churn rate

---

## 8. Data Model (High-Level)

- Users
- Subscriptions
- Plans
- Targets
- FollowersState
- FollowingState
- Events (90-day retention)
- DailyStats (permanent)
- SyncRuns
- EmailLog
- RequestUnitUsage

---

## 9. Error Handling

### Target Statuses

- Active
- Paused – Private
- Paused – Out of Plan
- Paused – Error
- Syncing

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

1. Exact follower count thresholds per tier?
2. Exact cooldown duration?
3. Daily sync limits per tier?
4. Page cap per sync?
5. Timezone handling for weekly emails?

---

**End of Document**
