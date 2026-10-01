# 14 — Billing, Demo Tier & Feedback

> _Owns: the credit system (1 credit = 100,000 tokens), the org spend-alert
> odometer, the public demo tier (self-signup on the standard free 5M budget +
> email verification + IP gating), feedback-for-tokens, and the "book a demo"
> funnel._
> _Changed 2026-07-06 (migration 018): **credits re-denominated to 1 credit =
> 100,000 tokens** (1,000,000 tokens = 10 credits) — balances/ledger/holds/plan
> grants are on the CREDIT scale; `usage_events` stays raw tokens. Added an
> **org spend-alert odometer** (notification-only 1×/2×/3× threshold; paid-org
> members are never blocked). See "Org spend-alert threshold" below._
> _Changed 2026-06-15: simplified to two user types — free + paid. Public demo
> signups are now plain free users (flat 5M, monthly); the separate one-time demo
> budget was removed (migration 011). `is_demo` is retained only for IP gating +
> the verify-to-keep purge._
> _Added 2026-06-13. Source of truth: [app/db/queries/credits.py](../app/db/queries/credits.py),
> [app/db/queries/org_alerts.py](../app/db/queries/org_alerts.py),
> [app/billing/](../app/billing/), [app/config.py](../app/config.py) (billing block),
> migrations [005](../migrations/005_credit_subscriptions.sql) / [006](../migrations/006_free_tier_tokens_feedback.sql) / [007](../migrations/007_demo_signup.sql) / [018](../migrations/018_credit_redenomination.sql)._

## The model in one paragraph

Usage is measured in **tokens** but **denominated in credits** — **1 credit =
100,000 tokens** (`tokens_per_credit`; 1,000,000 tokens = 10 credits, migration
018). Balances, the ledger, holds and plan grants are stored on the **credit**
scale; only `usage_events` keeps raw tokens. Token estimates/measurements are
divided by `tokens_per_credit` at the reserve/settle boundaries. Each metered
operation (chat, search, ingest, report) **reserves** an estimate up front under
a row lock, then **settles** against the **measured** tokens it actually
consumed — the same counts written to `usage_events` (see
[03](03-ingestion-pipeline.md)/[04](04-retrieval-and-chat.md) for where tokens
are produced). A user (or paid org) owns a token pool + monthly cycle; the
**free** tier is the trial gate. There are just two user types: **free** (flat
5M monthly — both invited free users and public self-signup/'demo' users, which
are identical except demo signups are IP-gated + verify-to-keep) and **paid**
(pro/unlimited — not enforced; consumption tracked in `usage_events`).

## Plans & tiers

Plans live in `subscription_plans` (seeded by migration 006). `monthly_credit_grant`
is stored on the **credit** scale (migration 018 rescaled it; the tuning knob
`free_monthly_token_budget` is in **tokens** and is converted to credits at
startup by `sync_free_plan_budget`). Per-feature call caps were dropped (the
budget is the only limit).

**Who wins on `monthly_credit_grant`.** Two write paths touch that one field:
the env knob at startup and `PUT /admin/plans/{key}` at runtime. Since migration
027, `subscription_plans.synced_from_config` records the config value startup
last wrote, and the precedence is:

| Situation | Result |
|---|---|
| Env knob changed since the last boot | Config wins — deliberate operator action |
| Env unchanged, admin edited via the API | **Admin edit is preserved** across restarts |
| `synced_from_config` is NULL (pre-027 row) | Adopted on the next boot; grant untouched |

Before 027 the startup sync overwrote the row unconditionally, so a super_admin
edit was silently reverted at the next restart and logged as `drift`.

**Opening an account.** Every signup path (invite, Google SSO, demo self-signup)
must create its `user_subscriptions` row through
`credits.new_user_subscription()`. `credit_balance` is `NOT NULL DEFAULT 0`, so a
hand-rolled row opens the account **empty** — and because it also opens a fresh
30-day cycle, `_renew_if_due` will not top it up either, leaving the user on
`402 insufficient_credits` for a full month. That was a live production bug in
the Google-SSO and invite paths (migration 026 backfilled the affected accounts).

| Plan | Grant | Notes |
|---|---|---|
| `free` | 50 credits (5,000,000 tokens) | invited B2B + demo self-signup; renews monthly |
| `pro` | — | paid; org pool; renews monthly |
| `unlimited` | — | `is_unlimited=true`; never blocked, metered for invoicing |

> **Credit unit:** 1 credit = 100,000 tokens, so the free 5M-token budget is
> **50 credits**. Displayed balances are credits; `/me/usage` `balance`/`limit`
> stay in **tokens** for the FE meter.

Effective tier = `max(user plan, org tier)` (`resolve_effective_tier`,
[credits.py](../app/db/queries/credits.py)). The pool debited is the **org** pool
when the user is in a paid org, else the user's own pool.

## Public demo tier (self-signup)

> Invited B2B stays invite-only ([02](02-auth-and-rbac.md)). The demo is a
> **separate, public** path so a landing-page visitor can self-register and try
> the product.

A demo user is **just a free user** with `user_subscriptions.is_demo=true` and
`users_profile.signup_source='demo_self_signup'` — same renewing **5M monthly**
budget as any free user (`free_monthly_token_budget`); there is **no separate
demo budget**. `is_demo` only marks them for the IP layer + verify-to-keep purge.
Differences from an invited free user:

- **Same 5M monthly budget**, renewed by `_renew_if_due` like any free user. When
  exhausted → `402` carrying `book_demo_hint:true`.
- **Immediate access (verify-in-background).** Created with
  `users.email_confirmed=false` but signup returns tokens — the user is in the
  sandbox at once. Verification is a **soft gate** (`demo_immediate_access`,
  default on): unverified → **no feedback bonus** + **purged after
  `demo_unverified_ttl_days`**. (Set the flag off to restore the old
  `403 email_not_verified` hard gate.)
- **IP-gated** (see below).

### Lifecycle

1. `POST /auth/demo/signup` → IP signup-cap check → create `users`
   (`email_confirmed=false`, hashed `verification_token_hash`) + `users_profile`
   (`signup_source='demo_self_signup'`) + `user_subscriptions` via
   `credits.new_user_subscription()` (`is_demo`, `credit_balance` = the `free`
   plan's `monthly_credit_grant`, 30-day cycle, `signup_ip`)
   → email a verification link →
   **`201` with a token bundle (`status:"active_unverified"`, `email_verified:false`)
   — auto sign-in.**
2. `POST /auth/verify-email {token}` → matches the SHA-256 token hash, checks
   `demo_verification_ttl_seconds`, sets `email_confirmed=true`, clears the token,
   returns a token bundle (`email_verified:true`) — cancels the purge + unlocks
   the feedback bonus.
3. `POST /auth/demo/resend-verification {email}` → re-issues a token (always `202`).
4. **Stale cleanup:** `app.tools.purge_demo` / the `cleanup.purge_stale_demo`
   Celery task (daily beat) deletes unverified demo accounts + their data after
   `demo_unverified_ttl_days`.

A demo user's documents/chats/projects/memories are stored exactly like any
user — isolated by `user_id` in the **default** Qdrant collections (no org), see
[05](05-storage-and-vectors.md). They only ever see their own personal projects.

## IP abuse-control layer

Because anyone can self-register, demo value is gated by the account's one-time
budget **and** by IP (so one person can't farm tokens across many accounts).
Redis-backed, **fail-open** (an outage never blocks legit traffic). Built on
[app/utils/rate_limit.py](../app/utils/rate_limit.py) + shared client-IP helper
[app/utils/net.py](../app/utils/net.py).

| Guard | Where | Key | Limit / window |
|---|---|---|---|
| Signup cap | `enforce_demo_signup_quota` (auth signup) | `rl:demo:signup:{ip}` | `demo_signup_max_per_ip` / `demo_signup_window_seconds` |
| Resend cap | `check_and_increment` (resend) | `rl:demo:resend:{ip}` | same as signup |
| Per-IP usage | `enforce_demo_ip` ([app/billing/demo_guard.py](../app/billing/demo_guard.py)) on chat/search/ingest/report | `rl:demo:use:{ip}` | `demo_ip_request_limit` / `demo_ip_window_seconds` |
| Booking spam | `enforce_demo_request_quota` (`/demo/request`) | `rl:demo:request:{ip}` | `demo_request_max_per_ip` / day |

`enforce_demo_ip(feature)` is attached to the metered routes via the route
**decorator** (`dependencies=[…]`), so it runs **before** the credit reserve and
**only** for demo users (`AuthUser.is_demo` + `tier=="free"`). Paid/invited users
skip it entirely. Over-limit → `429 {error_code:"demo_ip_limit", book_demo:true}`.

> SECURITY: `client_ip()` trusts the leftmost `X-Forwarded-For`. In production,
> terminate at a trusted proxy / run uvicorn with `--forwarded-allow-ips` so the
> header can't be spoofed to dodge limits. See [08](08-deployment-and-ops.md).

## Enforcement & settlement

[app/billing/deps.py](../app/billing/deps.py) `require_credits(feature)` reserves
the per-feature token estimate ([app/billing/estimates.py](../app/billing/estimates.py)
`estimate_tokens`) at the route entry; the route's settle/release point trues it
up. Reserve/settle/release live in [credits.py](../app/db/queries/credits.py).

- **Enforcement gate**: `enforced = billing_enforcement_enabled OR (free tier AND
  free_tier_enforced)`. So the **free tier is gated independently** of the global
  switch; paid tiers are gated only when `billing_enforcement_enabled=true`.
  `unlimited` never blocks. On insufficiency → `402` with `error_code`,
  `balance`, `required`, `upgrade_hint`, `earn_more_hint:"feedback"` (free), and
  `book_demo_hint` (demo).
- **Paid-org soft-cap (migration 018)**: members of a **paid** (`pro`/`unlimited`)
  org are **never interrupted** on balance — the org pool is allowed to go
  **negative**. The `insufficient` balance gate in `reserve_credits` is skipped
  for the org pool (per-feature caps still apply); instead the org **spend-alert
  odometer** notifies the org admin (see below). Self-served `free`/`payg` users
  keep the hard 402 gate. `super_admin` is always unlimited/exempt.
- **Settle on measured tokens** (not the estimate): chat sums guard+synth tokens
  off the response/stream-done meta (`_tokens_from_meta` in
  [chat.py](../app/routes/chat.py)); ingest/report pass `total_tokens` to
  `settle_by_ref`, which **divides by `tokens_per_credit`** before truing up;
  search settles ~0. Over-reservation is refunded, shortfall clawed back (user
  pools clamp ≥ 0; **org pools may go negative** so true overspend shows).
- **Fail-open**: any infra error in reserve returns a permissive hold; settle /
  release never raise. Billing must never take down the app.
- **Audit**: every movement is appended to `credit_ledger`; measured token/cost
  also lands in `usage_events` ([11](11-memory-and-chatgpt-roadmap.md) / admin
  usage dashboard).
- **Proactive balance**: `GET /me/usage` (`get_usage_summary`,
  [app/routes/me.py](../app/routes/me.py)) returns the caller's `balance` / `limit`
  / `unlimited` / `is_demo` / `caps` read-only, so the FE shows credits before a
  402. Exempt roles + unlimited tier report `unlimited:true` (no numeric limit);
  free (including demo signups) reports the monthly grant (5M tokens). For
  paid-org members it also returns the informational odometer fields
  `org_consumed_credits` / `org_threshold_credits` / `org_alert_tier` (0–3).

## Org spend-alert threshold (migration 018 — notification only)

Paid-org members are never blocked, so overspend is surfaced by an **org-wide
cumulative odometer** instead of a gate. Source of truth:
[credits.py](../app/db/queries/credits.py) (`_advance_org_odometer`,
`_dispatch_org_alert`) + [org_alerts.py](../app/db/queries/org_alerts.py).

- **Odometer**: `organizations.consumed_credits_cycle` accumulates every settled
  charge for the org this cycle. When it crosses **1× / 2× / 3×** the threshold
  it advances `organizations.alert_tier_fired` and emails the org's **admins
  only** (role `admin` in the same org) once per tier. After 3× it goes quiet
  until the next cycle; both fields **reset on cycle renewal** (`_renew_if_due`).
- **Threshold**: `organizations.alert_threshold_credits` (per-org override);
  `NULL` falls back to the global `org_alert_threshold_credits` (default **100
  credits**). `org_alerts_enabled` (global) and `organizations.alerts_enabled`
  (per-org) are kill switches — when off the odometer stops advancing and no
  alert fires.
- **Email**: `send_org_threshold_email` ([app/auth/email.py](../app/auth/email.py))
  includes a **per-user spend breakdown** (`org_usage_breakdown` — top spenders
  from `usage_events` since cycle start, tokens→credits). Dispatch is
  **fire-and-forget** after commit; the odometer state is already durable so a
  failed email doesn't lose the milestone.
- **super_admin config**: `PATCH /admin/organizations/{id}` accepts
  `alert_threshold_credits` (>0) and `alerts_enabled` — no new endpoint. See
  [07-api-reference.md](07-api-reference.md).

## Feedback-for-tokens (one-time)

`POST /feedback {rating 1–5, message}` ([app/routes/feedback.py](../app/routes/feedback.py)).
A submission meeting the quality bar (`message` ≥ `feedback_min_chars`) grants
`feedback_bonus_tokens` non-expiring bonus tokens **once per user lifetime**
(`grant_feedback_bonus` counts prior bonus-granting `user_feedback` rows, capped
by `feedback_max_grants_lifetime`; **not** per-cycle) — **only if the user's
email is verified** (`AuthUser.email_verified`); an unverified demo user's
feedback is saved + emailed but `bonus_granted:false`
(`detail:"email_not_verified"`). The row is always saved; the team is emailed
(`send_feedback_email` → support inbox). Bonus tokens are spent **before** base
balance at reserve time and survive cycle renewal.

## Book a demo

`demo_booking_url` (config) is the external Calendly/Cal.com link the FE renders
as the CTA. `POST /demo/request {name, email, company?, message?}` (public,
IP-limited) records a `demo_requests` lead, emails the team
(`send_demo_request_email`), and returns `{ok, booking_url}`. No calendar API
integration (future).

## Config knobs

[app/config.py](../app/config.py) billing block (all env-tunable):

```
tokens_per_credit=100_000            # 1 credit = 100k tokens (migration 018)
billing_enforcement_enabled=False    free_tier_enforced=True
free_monthly_token_budget=5_000_000  default_subscription_tier="free"
token_estimate_chat=4_000  token_estimate_report=60_000
token_estimate_ingest_per_mb=8_000   token_estimate_search=200
credit_hold_sweep_ttl_seconds=172_800
org_alert_threshold_credits=100.0    org_alerts_enabled=True   # org spend alert (notification only)
feedback_bonus_tokens=250_000  feedback_min_chars=120  feedback_max_grants_lifetime=1
demo_verification_ttl_seconds=86_400   demo_booking_url=""   # demo = free 5M (no separate budget)
demo_immediate_access=True   demo_unverified_ttl_days=7   # verify-in-background + stale purge
demo_ip_enforced=True  demo_signup_max_per_ip=3  demo_signup_window_seconds=86_400
demo_ip_request_limit=200  demo_ip_window_seconds=86_400
demo_max_accounts_per_ip=5  demo_request_max_per_ip=5
```

## Things this page does not own

- Schema/columns for `subscription_plans`, `user_subscriptions`, `credit_ledger`,
  `user_feedback`, `demo_requests`, `users` verification cols →
  [06-database-schema.md](06-database-schema.md).
- Endpoint request/response contracts → [07-api-reference.md](07-api-reference.md).
- Sign-in / JWT / invite flow / RBAC → [02-auth-and-rbac.md](02-auth-and-rbac.md).
- Where tokens are produced/measured → [03](03-ingestion-pipeline.md) /
  [04](04-retrieval-and-chat.md); the usage dashboard →
  [11](11-memory-and-chatgpt-roadmap.md).
