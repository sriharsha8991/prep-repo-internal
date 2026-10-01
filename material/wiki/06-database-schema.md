# 06 — Database Schema

> _Changed 2026-06-07: rewritten for local Postgres. Supabase is gone — there
> are **no RLS policies**, no `auth.users`, no `handle_new_user` trigger, and no
> Supabase Realtime. Ownership is enforced in app code (see
> [02-auth-and-rbac.md](02-auth-and-rbac.md)). Source of truth:
> [migrations/](../migrations/) + [app/db/models.py](../app/db/models.py)._

## Migrations

| File | Adds |
|---|---|
| `init.sql` | Consolidated schema (local Postgres): `user_role` enum, `users`, `users_profile`, `projects`, `documents`, `jobs`, chat sessions, memories, support tickets, content-hash extraction cache, subscriptions/credits, `user_feedback`, `demo_requests`, `set_updated_at` trigger |
| `002_batch_lane.sql` | Batch-lane columns on `jobs` (`lane`, `batch_state`, `batch_meta`) |
| `003_usage_events.sql` | `usage_events` ledger (per-user token/cost tracking) |
| `004_organizations_shared_projects.sql` | `organizations`, `users_profile.organization_id`, `projects.organization_id` + `visibility` |
| `005_credit_subscriptions.sql` | `subscription_plans`, `user_subscriptions`, `credit_ledger`, org credit-pool columns |
| `006_free_tier_tokens_feedback.sql` | Re-denominates plan grants to **tokens** + `user_feedback` (feedback-for-tokens) |
| `007_demo_signup.sql` | Public demo: `users` verification cols, `users_profile.signup_source`, `user_subscriptions.is_demo`/`signup_ip`, `demo_requests` |
| `008_report_blackboard.sql` | Agentic report `blackboard` + `quality_score` JSONB |
| `009_documents_org_id.sql` | Denormalized document organization id |
| `010_rescale_token_balances.sql` | Token-balance scale correction |
| `011_demo_is_free.sql` | Demo users normalized to free-tier semantics |
| `012_credit_holds.sql` | Explicit credit hold state machine |
| `013_usage_observability.sql` | Request/trace/org/project/document/session/job/report correlation on `usage_events` |
| `014_org_mandatory.sql` | Organization membership mandatory for non-super_admin |
| `015_project_members.sql` | Per-user project grants (`project_members`) |
| `016_payg_activation.sql` | PAYG org activation (`activated_at`, `overage_allowed`) |
| `017_super_admin_org.sql` | super_admin organization handling |
| `018_credit_redenomination.sql` | **Re-denominate credits to 1 credit = 100k tokens** (rescale balances/ledger/holds/plan grants) + org spend-alert columns (`consumed_credits_cycle`, `alert_tier_fired`, `alert_threshold_credits`, `alerts_enabled`) |
| `026_backfill_zero_credit_balances.sql` | Data-only repair: raises every `user_subscriptions.credit_balance` to its plan's monthly grant (top-up only; skips zero-grant plans so purchased PAYG credits survive). Fixes accounts opened at 0 by the Google-SSO / invite paths before they seeded the grant |
| `027_plan_grant_config_sentinel.sql` | `subscription_plans.synced_from_config` — lets startup tell a deliberate `PUT /admin/plans/{key}` apart from an env-knob change, so super_admin's edit is no longer reverted on every restart |

> `019`–`025` are schema migrations not yet itemized above; read the files.
> Alembic is the migration authority; the numbered SQL files above are mirrored
> as Alembic revisions (latest revision 007). Migration 018 uses a shared
> sentinel row (`schema_migrations.'redenomination_credits_v1'`) so whichever of
> the raw-SQL or Alembic path runs first claims the rescale — it never
> double-rescales.

`init.sql` is applied automatically by the Postgres container's initdb hook on a
fresh volume. Alembic is the migration authority for existing databases. The
first Alembic revision adopts the existing numbered SQL files and records the
database state in `alembic_version`; future schema changes should be new Alembic
revisions. Both production and local compose define a one-shot `migrate` service
that runs `alembic upgrade head` before app/workers start, so normal
deploy/startup applies pending migrations automatically.

```powershell
docker compose run --rm migrate
alembic current
alembic history
```

> This page summarises the core tables. For exact, current columns (including
> the support/chat/memory/usage tables) read `migrations/init.sql` and
> `app/db/models.py` — those are authoritative.

## Enums

```
public.user_role = (
  super_admin, admin,
  drilling_engineer, geologist, asset_manager, qa_steward
)
```

## Tables

> Identity is split across **two** tables. `users` is the credential store
> (email + password hash + verification state); `users_profile` holds the
> role/org/quota metadata. They share the same `id`. Source of truth:
> [app/db/models.py](../app/db/models.py) (`User`, `UserProfile`).

### users

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | account id; `default gen_random_uuid()` |
| `email` | text | unique; set at invite / demo-signup time |
| `password_hash` | text | bcrypt hash (auth is local now, not Supabase) |
| `email_confirmed` | bool NOT NULL DEFAULT true | invited B2B users start confirmed; demo self-signups start `false` until they confirm the emailed link |
| `verification_token_hash` | text | sha256 of the email-verification token (demo only; null for invited); cleared once confirmed |
| `verification_sent_at` | timestamptz | when the verification link was issued (demo only) |
| `created_at` / `updated_at` | timestamptz | default now() |

### users_profile

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | same id as the `users` row |
| `email` | text | mirror of `users.email` (denormalized) |
| `display_name` | text | nullable |
| `role` | `user_role` | default `drilling_engineer` |
| `organization` | text | **DEPRECATED** legacy display string; reads join `organizations.name` via `organization_id`. `PATCH /auth/me/profile` drops writes to it |
| `organization_id` | uuid | FK to `organizations`, ON DELETE SET NULL |
| `memory_enabled` | bool NOT NULL DEFAULT true | per-user cross-session memory toggle |
| `user_quota` | int NOT NULL DEFAULT 0 | meaningful only for admins; 0 = unlimited |
| `invited_by` | uuid | inviter's id (null for super_admin / pre-existing) |
| `signup_source` | text NOT NULL DEFAULT `invite` | `invite` (B2B) or `demo_self_signup` |
| `created_at` | timestamptz | default now() |
| `updated_at` | timestamptz | maintained by `set_updated_at` trigger |

Triggers: only `set_updated_at` (before update) bumps `updated_at`. There is
**no** `handle_new_user`/`on_auth_user_created` trigger — the invite handler
inserts the `users` + `users_profile` rows directly. **No RLS** —
`/auth/me/profile` and admin endpoints enforce ownership in app code.

### projects

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | default `gen_random_uuid()` |
| `user_id` | uuid | owner, ON DELETE CASCADE |
| `name` | text NOT NULL | |
| `description` | text | nullable |
| `created_at` / `updated_at` | timestamptz | |

Ownership enforced in app code (no RLS).

### documents

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | **stable across re-ingestions** |
| `user_id` | uuid | FK |
| `project_id` | uuid | FK to `projects`, ON DELETE CASCADE |
| `filename` | text | display name |
| `artifact_class` / `confidentiality` | text | governance metadata (PATCHable) |
| `content_hash` | text | sha256 of the PDF (dedup / cache key) |
| `superseded_by` | uuid | newer revision, nullable |
| `created_at` / `updated_at` | timestamptz | |

Ownership enforced in app code (no RLS).

### jobs

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | celery task id |
| `document_id` | uuid | FK |
| `user_id` | uuid | FK |
| `status` | text | `queued | running | succeeded | failed` |
| `current_phase` | text | free-form; UI shows it as-is |
| `pages_total` / `pages_done` | int | progress meter |
| `error` | text | truncated traceback on failure |
| `lane` | text | `realtime` \| `batch` (002_batch_lane) |
| `batch_state` / `batch_meta` | text / jsonb | batch-lane bookkeeping (002_batch_lane) |
| `created_at` / `started_at` / `ended_at` | timestamptz | |

Ownership enforced in app code (no RLS). **No Supabase Realtime** — the FE
polls `GET /ingest/{job_id}`.

### Subscriptions, credits, feedback & demo (migrations 004–007)

Behaviour/semantics are owned by [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md);
this page owns the columns. Balances are **token**-denominated.

- **`subscription_plans`** — `key` (`free|pro|unlimited`), `monthly_credit_grant`
  (**credits** since migration 018; 1 credit = 100k tokens), `is_unlimited`,
  `per_feature_caps` (now `{}`), `credit_cost_multiplier`.
- **`user_subscriptions`** — one per user: `plan_key`, `credit_balance` (**credits**),
  `bonus_credits` (non-expiring, credits), `feature_usage` (jsonb), `cycle_start/end`,
  `organization_id`, **`is_demo`** (one-time budget; skips renewal), **`signup_ip`**.
- **`credit_ledger`** — append-only audit (`reserve|settle|release|bonus|renew_reset|adjust|threshold_alert`); credit-scaled `credits_delta`; `ref_id` joins `usage_events`.
- **`user_feedback`** — `user_id`, `rating`, `message`, `bonus_tokens_granted`
  (>0 marks a lifetime grant; gates one-time feedback bonus).
- **`demo_requests`** — `name`, `email`, `company?`, `message?`, `source_ip`
  (public "book a demo" leads).
- **`users`** adds `verification_token_hash`, `verification_sent_at` (demo email
  verification; `email_confirmed` starts `false` for demo signups).
- **`users_profile`** adds `signup_source` (`invite` | `demo_self_signup`).
- **`organizations`** (migration 004) carries its own credit-pool columns
  (`tier`, `credit_balance`, `bonus_credits`, `cycle_*`, `activated_at`,
  `overage_allowed`) for paid-org pooling. Migration 018 adds the **spend-alert
  odometer**: `consumed_credits_cycle` (running cumulative spend this cycle),
  `alert_tier_fired` (0–3, last threshold multiple notified), `alert_threshold_credits`
  (per-org override; `NULL` → global `org_alert_threshold_credits`) and
  `alerts_enabled` (per-org kill switch). `consumed_credits_cycle`/`alert_tier_fired`
  reset at cycle renewal.

## Auth gate (no RLS)

There are no RLS policies. Every read/write goes through the app, which filters
by `user_id` (and `organization`/`project_id` where relevant) and checks
ownership in the route (`_resolve_doc`, `current_user`, `require_role`).
super_admin uses explicit `?scope=all` rather than a silent bypass.

## Schema vs. ORM

[app/db/models.py](../app/db/models.py) is the SQLAlchemy 2.0 mapping.
It must mirror the SQL exactly (column types, enum values, defaults).
When you change a migration, change the ORM in the same PR. The
`ROLE_VALUES` tuple in `models.py` is asserted to match the SQL enum
in the test suite.

## Things this page does not own

- Local storage layout → [05-storage-and-vectors.md](05-storage-and-vectors.md).
- Auth tier semantics → [02-auth-and-rbac.md](02-auth-and-rbac.md).
- How to apply / roll back migrations → [10-runbooks.md](10-runbooks.md).
- Credit/token-budget, demo & feedback behaviour → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).
