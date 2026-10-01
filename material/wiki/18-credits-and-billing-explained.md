# 18 — Credits & Billing, Explained

> _This page is a plain-English walkthrough of the credit/billing system as it
> works **today** (2026-07-27). It is a companion to
> [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md), which
> stays the terse, dated reference. Read this page first if you're new to the
> system; use 14 when you need the exact contract for a specific field or
> error code. Source of truth for behaviour:
> [app/db/queries/credits.py](../app/db/queries/credits.py),
> [app/routes/admin_credits.py](../app/routes/admin_credits.py),
> [app/routes/admin_organizations.py](../app/routes/admin_organizations.py),
> [app/routes/feedback.py](../app/routes/feedback.py),
> [app/routes/me.py](../app/routes/me.py), [app/db/models.py](../app/db/models.py)._

## 1. The one-paragraph mental model

Every user (and every paid organization) owns a **token pool** that refills
once a month. Using the product — chatting, ingesting a document, generating
a report, searching — spends tokens out of that pool. Free users have a hard
ceiling (5,000,000 tokens/month); once it's gone, they're blocked until the
next cycle or until they earn more by leaving feedback. Paid users
(pro/payg/unlimited) are essentially never blocked — their usage is tracked
for invoicing, not gated. `super_admin` never sees any of this. Balances are
stored internally as **credits**, where **1 credit = 100,000 tokens** — that's
just a display/storage scale factor, not a different currency.

## 2. Vocabulary you need before anything else makes sense

| Term | Meaning |
|---|---|
| **Token** | The unit the LLM actually bills in. What the frontend shows the user. |
| **Credit** | The unit balances are *stored* in. `1 credit = 100,000 tokens` (`settings.tokens_per_credit`). A 5,000,000-token free budget is stored as `50` credits. |
| **Pool** | The account that gets debited — either a user's own `user_subscriptions` row, or (if the user belongs to a paid org) that org's `organizations` row. |
| **Cycle** | A rolling 30-day window (`cycle_start` → `cycle_end`) after which the pool's balance resets to the plan's grant (for grant-funded plans) and per-feature counters clear. |
| **Hold** | A temporary reservation created before an operation runs, so two concurrent requests can't overspend the same balance. Every hold is finalized exactly once — `settled` or `released`. |
| **Ledger** | `credit_ledger` — an append-only audit trail of every balance movement. This is the "bank statement"; the balance columns are just a cached running total. |
| **Bonus credits** | Non-expiring credits (from feedback, or an admin grant) that are always spent *before* the regular cycle balance, and survive cycle renewal. |
| **Effective tier** | `max(user's own plan, their org's tier)` — whichever is higher wins. |

## 3. The two kinds of user, in practice

| | Free | Paid (pro / payg / unlimited) |
|---|---|---|
| Monthly budget | 5,000,000 tokens (50 credits), renews every cycle | pro/unlimited: not enforced at all. payg: no monthly grant — funded only by top-ups. |
| What happens at zero | **Hard block** — `402 insufficient_credits` | pro/unlimited: never blocked. payg: blocked unless the org has `overage_allowed`. |
| Who's on this tier | Every invited B2B user by default, **and** every public demo self-signup (they're the *same* thing — see §7) | Users attached to an org whose `tier` is `pro`/`payg`/`unlimited`, or a user whose *individual* `plan_key` was set to one of those |
| Can earn more | Yes — one-time feedback bonus (§6) | N/A (not gated) |

`super_admin` is a third, invisible category: `BILLING_EXEMPT_ROLES = {"super_admin"}`
in [credits.py](../app/db/queries/credits.py) makes every reserve short-circuit to
"unlimited, not enforced" **before touching the database at all**, regardless
of whatever subscription row that account happens to have.

## 4. Where the plan definitions live

`subscription_plans` is a 4-row table (`free`, `payg`, `pro`, `unlimited`) —
the single source of truth for every tier's monthly grant, per-feature caps,
and whether it's unlimited. A user or org doesn't carry its own grant amount;
it just carries a `plan_key`, and the actual number is looked up from this
table at reserve time. That indirection is what lets a super_admin change
"how many tokens does `free` get" for everyone by editing one row, instead of
touching every user.

```
subscription_plans
├── free       monthly_credit_grant = 50   (= 5,000,000 tokens)   is_unlimited = false
├── payg       monthly_credit_grant = 0    (funded only by top-ups)
├── pro        monthly_credit_grant = <configured>                is_unlimited = false
└── unlimited  monthly_credit_grant = 0                            is_unlimited = true
```

**Who is allowed to change the `free` grant, and how the two writers don't
stomp on each other:** the env var `free_monthly_token_budget` is pushed into
the `free` row at every app startup (`sync_free_plan_budget`), so ops can
change it just by editing config and restarting. But a super_admin can *also*
edit it live via `PUT /admin/plans/free`. Before migration 027, startup
blindly overwrote the row on every boot, so a super_admin's live edit got
silently reverted at the next restart. The fix: `subscription_plans.synced_from_config`
remembers what config value startup *last* pushed. Now:

- If the env var changed since last boot → config wins (an operator made a
  deliberate change).
- If the env var is unchanged but the row's value differs from what startup
  last wrote → that's an admin edit, and it's **left alone**.

## 5. Individual pool vs. org pool — who actually pays

A user's `user_subscriptions` row is *always* the place per-feature counters
live. But the balance that actually gets debited depends on organization
membership:

- **Solo user, or user in a `free`-tier org** → their own `user_subscriptions.credit_balance`
  is debited.
- **User in a `pro`/`payg`/`unlimited` org** → the **org's** `organizations.credit_balance`
  is debited instead. Every member of that org draws from the same shared
  wallet.

This is why an org has its own `credit_balance`, `bonus_credits`, and
`cycle_start`/`cycle_end` columns that mirror `user_subscriptions` — it's a
second kind of pool, at the tenant level instead of the individual level.

### The "soft cap" philosophy for paid orgs

Ship 7 made a deliberate product decision: **members of a paid org are never
interrupted by a 402 above zero.** If the org's wallet has *any* positive
balance, the debit goes through even if it's not enough to fully cover the
estimate — the wallet is allowed to dip to exactly zero rather than block the
request mid-task. Only once the wallet is at or below zero does a hard block
kick in, and even that can be waived per-org via `overage_allowed` (lets the
balance go negative, billed later). A super_admin can restore the old
hard-cap behaviour for one specific tenant by flipping
`organizations.soft_cap_never_block` to `false`.

Instead of blocking, overspend on a paid org is surfaced via a notification —
see §9.

## 6. Walking through a real request: reserve → settle

Every metered feature (chat, search, ingest, report) follows the same
two-step pattern. Using a chat message as the example:

1. **Reserve (before the LLM call runs).** The route asks
   `reserve_credits(feature="chat", estimate=4000, ...)`. This:
   - Locks the user's `user_subscriptions` row (and the org row too, if they're
     in a paid org) with `SELECT ... FOR UPDATE`, so two simultaneous requests
     from the same user can't both read the same balance and both proceed.
   - Renews the pool first if its cycle has expired (see §8).
   - Checks: is this tier enforced? Is there enough balance (or, for a
     soft-capped org, *any* positive balance)?
   - If insufficient and enforced → raises `402` immediately, nothing is
     charged.
   - Otherwise, debits the **estimate** (4,000 tokens ≈ 0.04 credits) from
     the pool, bumps the per-user `feature_usage` counter, writes a `reserve`
     row to `credit_ledger`, and writes an **open** row to `credit_holds` —
     the receipt for what happens next.
2. **The LLM call runs.** Tokens are actually consumed — usually a different
   number than the 4,000-token estimate.
3. **Settle (after the response).** The route calls `settle_credits` (or, for
   async workers like ingest/report, `settle_by_ref`) with the **measured**
   token count. This claims the hold (flips `open` → `settled`, and only the
   first caller to do this wins — a retry or a race is a no-op), then
   true-ups the balance: refunds the difference if the estimate was too high,
   claws back more if it was too low. A `settle` row lands in `credit_ledger`.

If step 2 fails outright (the LLM call errors, the job crashes), the route
calls `release_hold` instead of `settle_credits` — that refunds the **entire**
reservation, as if it never happened.

**Why the hold exists at all, not just a straight debit-then-refund:** a
worker can be killed mid-job (OOM, redeploy, crash) after the reserve but
before it ever calls settle or release. Without a persisted hold, that
reservation would sit there debited forever with no way to know it needs a
refund. Because the hold is a real row, a background sweep
(`sweep_orphaned_holds`, on a timer via `credit_hold_sweep_ttl_seconds`)
finds any hold still `open` after 48 hours and releases it automatically.

## 7. What a blocked user actually sees

When `reserve_credits` decides to block, it raises `402` with a JSON body
that's designed to be directly renderable by the frontend:

```json
{
  "error_code": "insufficient_credits",
  "feature": "chat",
  "tier": "free",
  "balance": 12.5,
  "required": 40.0,
  "cycle_end": "2026-08-15T00:00:00Z",
  "upgrade_hint": "pro",
  "earn_more_hint": "feedback",
  "book_demo_hint": false
}
```

- `upgrade_hint` tells the FE which tier to pitch (`pro` for a blocked free
  user, `top_up` for a blocked payg user, `unlimited` otherwise).
- `earn_more_hint: "feedback"` only appears for free users — it's the cue to
  show "leave feedback for more tokens" (§9).
- `book_demo_hint` is `true` only when the blocked user is a demo self-signup
  — a sales signal to nudge them toward booking a call, not a different
  budget.
- The other possible `error_code` is `feature_cap_exceeded` (a plan's
  `per_feature_caps` limit was hit) or `org_not_activated` (see §11) — that
  one fires even before the balance check, and independently of whether
  billing enforcement is globally on.

Before any of this ever surprises a user, `GET /me/usage` lets the frontend
show their balance/limit proactively (a meter in the UI), including — for
paid-org members — the org's spend-alert odometer fields described in §9.

## 8. Cycle renewal — what "monthly" actually means

There's no cron job that resets every account on the 1st of the month.
Instead, every pool row (`user_subscriptions` or `organizations`) carries its
own `cycle_start`/`cycle_end`, and renewal happens **lazily**, the next time
that pool is touched by a reserve: if `now >= cycle_end`, the pool's cycle is
advanced (possibly by more than one 30-day window, if it's been untouched for
a while) and:

- **Grant-funded plans** (`free`, `pro` — positive `monthly_credit_grant`) get
  their balance **overwritten** with the fresh grant. Use-it-or-lose-it.
- **Zero-grant plans** (`payg`, `unlimited`) **carry the balance forward** —
  overwriting it would destroy credits the customer actually paid for. Only
  the cycle window advances.
- Bonus credits are **never** touched by renewal — they're a separate pot
  that only grows (feedback, admin grants) and only shrinks by being spent.
- For a user pool, `feature_usage` (the per-feature-per-cycle call counters)
  resets to `{}`.
- For an org pool, the spend-alert odometer (`consumed_credits_cycle`,
  `alert_tier_fired`) also resets, so the 1×/2×/3× emails can fire again next
  cycle.

## 9. The org spend-alert odometer — notification, never a block

Since paid-org members are never hard-blocked (§5), overspend needs a
different signal: a running total. Every settled charge against an org's pool
adds to `organizations.consumed_credits_cycle`. When that cumulative total
crosses 1×, 2×, or 3× a threshold, the org's admins (and every super_admin,
so the platform team always sees it too) get an email plus an in-app
notification. After 3× it goes quiet until the cycle renews.

The threshold itself can be set two ways per org (`alert_threshold_credits`,
an absolute number, or `alert_threshold_pct`, a percent of the org's total
cycle funding) — whichever produces the smaller number fires first. If the
org sets neither, it falls back to the global default
(`org_alert_threshold_credits`, 100 credits). A super_admin can silence the
whole mechanism per-org (`alerts_enabled=false`) or globally
(`org_alerts_enabled=false`).

A super_admin top-up (§11) clears the alert tier back to 0 (so the FE banner
disappears) without touching the cumulative odometer — only a real cycle
renewal zeroes that.

## 10. Feedback → bonus tokens (one-time, ever)

`POST /feedback {rating, message}` — anyone can submit feedback, but a
submission only earns bonus tokens if:

1. The message is at least `feedback_min_chars` (120) characters — a quality
   bar so a one-character note can't farm tokens.
2. The user's **email is verified**. An unverified public demo signup can
   still submit feedback (it's saved, the team is emailed), but gets
   `bonus_granted: false, detail: "email_not_verified"` — this is the "verify
   your email to unlock rewards" incentive.
3. They haven't already received this bonus before — it's capped at
   `feedback_max_grants_lifetime` (default **1**), counted across their
   *entire account history*, not per cycle.

A successful grant adds `feedback_bonus_tokens` (250,000 tokens = 2.5
credits) to `bonus_credits`, which — being bonus credits — never expires and
is spent before the regular balance.

## 11. Organizations: activation, PAYG, and top-ups

Creating an org (`POST /admin/organizations`, super_admin only) doesn't
automatically mean it can spend money. There's a deliberate activation gate:

- A brand-new org sits with `activated_at = NULL`. If its tier is `payg` or
  higher, **every credit-consuming feature 402s** with `org_not_activated` —
  members can log in and browse, but can't burn spend, until a super_admin
  explicitly calls `POST /admin/organizations/{id}/activate`. This check runs
  before the balance check and even before the global enforcement switch, so
  it can't be accidentally bypassed.
- Activation optionally seeds an initial wallet balance and sets
  `overage_allowed` (whether the PAYG wallet may go negative rather than
  hard-blocking at zero).
- After that, `POST /admin/organizations/{id}/top-up` adds more credits (or
  bonus credits) at any time. A top-up also clears the alert-tier odometer
  (§9) but keeps the running cumulative spend for the cycle.
- `PATCH /admin/organizations/{id}` is the one place to rename the org,
  change its tier, or tweak overage/alert settings — the org's `slug` (the
  physical Qdrant collection key) is **frozen at creation** and never
  changes, even if the display name does.

PAYG specifically has no monthly grant at all — its balance only ever comes
from top-ups, which is why it's *always* gated (no free monthly refill to
fall back on) unless `overage_allowed` is set.

## 12. What an admin/super_admin can actually do (API surface)

| Action | Endpoint | Who |
|---|---|---|
| Set a user's individual tier | `PATCH /admin/users/{id}/subscription` | super_admin (any user), admin (own org only) |
| Set an org's pool tier | `PATCH /admin/orgs/{id}/subscription` | super_admin only |
| Grant a user bonus credits | `POST /admin/users/{id}/credits` | super_admin, admin (own org) |
| View a user's live balance | `GET /admin/users/{id}/credits` | super_admin, admin (own org) |
| List all plans | `GET /admin/plans` | super_admin |
| Edit a plan's grant/caps/unlimited flag | `PUT /admin/plans/{key}` | super_admin |
| Create an org | `POST /admin/organizations` | super_admin |
| List / view orgs | `GET /admin/organizations`, `GET /admin/organizations/{id}` | super_admin |
| Rename org / change tier / overage / alerts | `PATCH /admin/organizations/{id}` | super_admin |
| Activate an org for billing | `POST /admin/organizations/{id}/activate` | super_admin |
| Add credits to an org wallet | `POST /admin/organizations/{id}/top-up` | super_admin |

Setting a tier (user or org) only ever **tops up** the balance if the new
plan's grant is higher than the current balance — it never claws back
mid-cycle credits the account already had.

## 13. Config knobs (env-tunable)

```
tokens_per_credit=100_000            # 1 credit = 100k tokens
billing_enforcement_enabled=False    # gates PAID tiers globally (free is separate — always on by default)
free_tier_enforced=True              # gates the free tier independently of the switch above
free_monthly_token_budget=5_000_000  # pushed into subscription_plans.free at every boot (unless admin-overridden, §4)
default_subscription_tier="free"

token_estimate_chat=4_000            token_estimate_report=60_000
token_estimate_ingest_per_mb=8_000    token_estimate_search=200

credit_hold_sweep_ttl_seconds=172_800   # 48h — orphaned-hold safety net

org_alert_threshold_credits=100.0    org_alerts_enabled=True

feedback_bonus_tokens=250_000  feedback_min_chars=120  feedback_max_grants_lifetime=1

demo_verification_ttl_seconds=86_400   demo_booking_url=""
demo_immediate_access=True             demo_unverified_ttl_days=7
demo_ip_enforced=True  demo_signup_max_per_ip=3  demo_signup_window_seconds=86_400
demo_ip_request_limit=200  demo_ip_window_seconds=86_400
```

## 14. Failure philosophy: billing must never take the product down

Every reserve/settle/release call is wrapped so that an infra error (DB down,
timeout) **fails open** rather than blocking a real user — reserve returns a
permissive "unlimited" hold on any unexpected exception, and settle/release
just log a warning and move on. The one thing that's never allowed to fail
silently is the *ledger*: every real movement that does succeed is recorded
in `credit_ledger`, which is the actual source of truth — the balance columns
on `user_subscriptions`/`organizations` are just a cached running total for
fast reads.

## 15. Common questions

**"Why did this user get a 402 even though `billing_enforcement_enabled` is
false?"** — That switch only controls *paid* tiers. The free tier has its own
independent switch, `free_tier_enforced` (defaults **on**). A free user is
gated regardless of the global switch.

**"A user is in a paid org — why did they still get blocked?"** — Either the
org itself hasn't been activated yet (`org_not_activated`, §11), or the
org's wallet is at/below zero and `overage_allowed` isn't set (the one case
where the soft-cap in §5 does still block).

**"I changed `free_monthly_token_budget` in the env and it didn't take
effect."** — Check whether a super_admin previously edited the `free` plan
via the API — if the *config value itself* hasn't changed since the last
boot, the admin's edit is deliberately preserved (§4). Changing the env value
to something new, then restarting, always wins.

**"What happens to a report job's reservation if the worker gets killed?"**
— It sits as an `open` hold until `sweep_orphaned_holds` finds it past the
TTL (48h default) and releases it as a refund. If the worker actually
finished and called settle around the same time, whichever one claims the
hold first wins — no double-refund, no double-charge.

## Things this page does not own

- Exact request/response schemas, error-code catalogue → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md) and [07-api-reference.md](07-api-reference.md).
- Table/column definitions → [06-database-schema.md](06-database-schema.md).
- How a user gets an account in the first place → [19-user-and-org-management-explained.md](19-user-and-org-management-explained.md).
- Sign-in / JWT mechanics → [02-auth-and-rbac.md](02-auth-and-rbac.md).
