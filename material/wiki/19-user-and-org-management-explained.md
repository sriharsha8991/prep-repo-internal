# 19 — User & Organization Management, Explained

> _This page is a plain-English walkthrough of how accounts, roles, and
> organizations (tenants) work **today** (2026-07-27). It complements
> [02-auth-and-rbac.md](02-auth-and-rbac.md) (sign-in/JWT mechanics) and fills
> a gap neither 02 nor 06 fully covers on its own: the day-to-day admin
> operations for managing users and organizations. Source of truth for
> behaviour: [app/routes/admin_users.py](../app/routes/admin_users.py),
> [app/routes/admin_organizations.py](../app/routes/admin_organizations.py),
> [app/auth/routes.py](../app/auth/routes.py), [app/db/models.py](../app/db/models.py)._

## 1. The identity model, in one picture

Every account is really **two rows sharing one id**:

```
users               ← credentials + verification state (email, password_hash,
  id (PK)              email_confirmed, verification_token_hash)
  email
  password_hash
  email_confirmed

users_profile        ← everything else about the account
  id (PK, = users.id)
  role                (which of the 6 roles below)
  organization_id     (which tenant they belong to)
  user_quota          (only meaningful if role == admin)
  invited_by          (who created this account)
  signup_source        'invite' | 'demo_self_signup'
```

Why split? `users` is the minimal thing auth needs to check a password and
know if the email is verified. `users_profile` is everything about *who this
person is in the product* — role, org, quota. Keeping them separate means the
auth layer never has to know about billing/org concepts, and vice versa.

There's a third row created alongside every new account:
`user_subscriptions` — the billing pool described in
[18-credits-and-billing-explained.md](18-credits-and-billing-explained.md).
**Every** account-creation path (invite, demo self-signup) must create all
three rows in the same transaction, or the account is left in a broken state
— see the callout in §3.

## 2. The six roles

| Role | Who creates it | What it can do |
|---|---|---|
| `super_admin` | Bootstrapped via CLI (`app.tools.bootstrap_super_admin`), or invited by another super_admin | Everything. Any user, any org, any tier, any role assignment. |
| `admin` | Invited by a super_admin (with an `organization` + `user_quota`) | **Org-scoped.** Invite/list/delete users in their own org only, up to their quota. Cannot create other admins or super_admins, cannot touch other orgs. |
| `drilling_engineer` | Invited by an admin or super_admin | End user. Own projects/documents only (or shared projects if their org is on a paid tier — [05-storage-and-vectors.md](05-storage-and-vectors.md)). |
| `geologist` | ″ | ″ |
| `asset_manager` | ″ | ″ |
| `qa_steward` | ″ | ″ |

The four end-user roles exist purely for UI/permission differentiation
inside a tenant (which dashboards/views they see) — none of them can invite
or manage other users. Only `admin` and `super_admin` are `INVITER_ROLES`.

An **admin's assignable-role list is deliberately narrower** than
super_admin's: `ADMIN_ASSIGNABLE_ROLES = {drilling_engineer, geologist,
asset_manager, qa_steward}` — an admin can never mint another admin or a
super_admin, even inside their own org. That has to go through a super_admin.

## 3. Two ways an account is created

**B2B is invite-only.** There is exactly one public self-service path (the
demo). Everything else goes through `POST /admin/users/invite`.

| Path | Who triggers it | Starting state |
|---|---|---|
| **Invite** | An admin or super_admin, via `POST /admin/users/invite` | `email_confirmed=true` (pre-confirmed — no verification step), random password emailed |
| **Demo self-signup** | Anyone, via public `POST /auth/demo/signup` | `email_confirmed=false`, but **immediate access** — signed in right away, verification happens in the background (see [02](02-auth-and-rbac.md) and [18](18-credits-and-billing-explained.md) §10 for the verify-to-earn-bonus / verify-to-keep-data angle) |

> **A real bug that shipped and was fixed (2026-07-27):** every account-creation
> path *must* build its `user_subscriptions` row through
> `credits.new_user_subscription()`, which seeds the balance from the plan's
> monthly grant. `credit_balance` defaults to `0` at the column level, so any
> path that hand-rolled the row instead opened the account **completely empty**
> — and because a fresh 30-day cycle also blocks the lazy-renewal logic, that
> user would sit at `402 insufficient_credits` for a full month with no
> obvious cause. This actually happened in the Google-SSO and invite paths;
> migration 026 backfilled the affected accounts. If you ever add a new
> way to create a user, route it through `new_user_subscription`.

## 4. The invite flow, step by step

`POST /admin/users/invite` ([app/routes/admin_users.py](../app/routes/admin_users.py)):

1. **Caller check.** Must be `admin` or `super_admin` (`_require_inviter`) —
   403 `requires_role` otherwise.
2. **Resolve the role.** Defaults to `drilling_engineer` if omitted. An admin
   caller trying to assign anything outside `ADMIN_ASSIGNABLE_ROLES` (i.e.
   `admin` itself) gets 403 `role_not_assignable_by_admin`.
3. **Resolve the organization** — every invitee must land in a real
   `Organization` row:
   - **Admin caller** → forced into the admin's own `organization_id`. If the
     admin somehow has no org, that's a data-integrity bug, not something to
     paper over → 409 `admin_missing_organization`.
   - **Super_admin caller** → must supply either `organization_id` (attach to
     an existing tenant, validated to exist up front) or `organization_name`
     (create-or-attach by name — if no org with that name/slug exists yet,
     one is created **as part of this same request**; if the slug collides
     with an existing org under a *different* display name, the request is
     refused with 409 `org_slug_conflict` rather than silently guessing which
     tenant was meant).
4. **Quota check** (admin callers only). Counts existing invitees
   (`users_profile.invited_by = caller.id`) against the admin's `user_quota`.
   `0` means unlimited. Over quota → 403 `quota_exceeded`.
5. **Duplicate check.** An existing user with that email → 409
   `email_already_registered`.
6. **Create the rows**, in one transaction: a random `secrets.token_urlsafe(12)`
   password (bcrypt-hashed) into `users`, the profile into `users_profile`
   (role, org, quota-for-invitee, `invited_by=caller.id`), and the billing
   pool via `new_user_subscription` (seeded from the assigned org's — or
   default — plan grant).
7. **Email the password** via Resend. **If the email fails to send, the whole
   invite is rolled back** — profile, subscription, and user rows are deleted
   explicitly (not relying on a DB cascade, since the ORM models don't declare
   one — see the note in admin_users.py). If this invite was also the one
   that *created* the organization, that org is deleted too, **but only if
   it's still empty** — a concurrent invite into the same brand-new org
   legitimately attached someone else in the meantime, and dropping the org
   would strand their membership.

The user signs in with the emailed temporary password and is expected to
change it via `POST /auth/change-password`.

## 5. Managing existing users

| Action | Endpoint | super_admin | admin |
|---|---|---|---|
| List / search users | `GET /admin/users` | any user, any org | own org only |
| Change a user's role | `PATCH /admin/users/{id}/role` | ✅ | ❌ (super_admin only) |
| Change an admin's invite quota | `PATCH /admin/users/{id}/quota` | ✅ | ❌ (super_admin only) |
| Delete a user | `DELETE /admin/users/{id}` | anyone (except self) | non-admin users in own org only |
| Resend invite (new temp password) | `POST /admin/users/{id}/resend-invite` | anyone (except self) | non-admin users in own org only |

Some guardrails worth calling out explicitly, because they're enforced with
dedicated error codes rather than just "403":

- **Nobody can delete or resend-invite themselves** — 400
  `cannot_delete_self` / `cannot_resend_self`.
- **An admin can never touch another admin or a super_admin**, even in their
  own org — 403 `cannot_delete_admin_or_super` / `cannot_resend_admin_or_super`.
  Role/quota changes are super_admin-only outright.
- **An admin is scoped to their exact `organization_id`** (not the legacy
  free-text `organization` string) — cross-org access is 403
  `cross_org_forbidden`.
- Listing supports `q` (substring match on email/display name), `role`
  filter, and pagination (`limit`/`offset`) — an admin's list is
  transparently pre-filtered to their org; they can't widen it.

Resend-invite does **not** touch role, org, or quota — it only regenerates
the password and re-sends the email. It's the tool for "this person never got
their invite email" or "reset this user's password," not for changing who
they are.

## 6. Organizations — the tenant layer

An `Organization` is the unit of multi-tenancy: it owns a Qdrant vector
collection (via a **frozen** `slug`, computed once from the name at creation
and never recalculated — renaming the org later only changes the display
`name`, not the physical storage key), a billing pool (§ below and
[18](18-credits-and-billing-explained.md)), and a set of member users.

| Action | Endpoint | Notes |
|---|---|---|
| Create an org | `POST /admin/organizations` | super_admin only. `name` + optional `tier` (defaults `free`). Rejects a name whose slug collides with an existing org (409 `org_slug_conflict`) or reduces to nothing alphanumeric (422). |
| List / search orgs | `GET /admin/organizations` | super_admin only. Substring on name/slug, tier filter, pagination. |
| View one org | `GET /admin/organizations/{id}` | super_admin only. |
| Rename / change tier / overage / alerts | `PATCH /admin/organizations/{id}` | super_admin only. Slug is immutable; everything else is editable. |
| Activate for billing | `POST /admin/organizations/{id}/activate` | super_admin only. Required before a `payg`+ org can spend — see [18](18-credits-and-billing-explained.md) §11. |
| Add credits to the org wallet | `POST /admin/organizations/{id}/top-up` | super_admin only. Refuses on an unactivated org (409 `org_not_activated`). |

Every write to an org that changes member-facing state (activation, top-up)
fires an in-app notification to **every member of that org** *and* every
super_admin — so the platform team's own notification bell is a single pane
of glass across every tenant, not something they have to check per-org.

**Membership is mandatory for non-super_admin users** (migration 014) — an
admin cannot exist without an `organization_id`, and every invitee (whether
attached to an existing org or a brand-new one) is assigned one at invite
time. There is no "default"/orgless bucket for regular accounts any more.

## 7. Permission model summary

Think of it as three concentric rings:

```
super_admin
  └─ sees / manages everything: any user, any org, any tier, plan config
admin
  └─ sees / manages only their own org's non-admin users
     cannot: create admins, cross into another org, change roles/quotas
drilling_engineer / geologist / asset_manager / qa_steward
  └─ no admin surface at all — just their own projects/documents
     (+ shared projects if their org is paid, see 05-storage-and-vectors.md)
```

Every admin endpoint enforces this at the route/query level (not via
database RLS — there is none, see [06](06-database-schema.md)) — a `WHERE
organization_id = caller.organization_id` clause for admins, and an explicit
`_assert_can_manage_user` / cross-org check before any mutation.

## 8. Sign-in, sessions, and password change (brief — owned by 02)

Full mechanics live in [02-auth-and-rbac.md](02-auth-and-rbac.md); the short
version:

- Login is `POST /auth/login {email, password}` → self-signed HS256 JWT
  access + refresh token pair. No Supabase, no JWKS.
- `POST /auth/change-password` is the only way a user changes their own
  password (requires the current password).
- Bootstrapping the very first `super_admin` on a fresh environment is a CLI
  tool, `app.tools.bootstrap_super_admin` — idempotent (re-running it on an
  existing email just upgrades that account's role).

## 9. Error codes cheat-sheet

| Code | HTTP | Meaning |
|---|---|---|
| `requires_role` | 403 | Caller's role isn't in the endpoint's allowed set |
| `role_not_assignable_by_admin` | 403 | An admin tried to invite someone as `admin`/`super_admin` |
| `admin_missing_organization` | 409 | An admin account has no `organization_id` (data bug) |
| `organization_required` | 422 | Super_admin invite omitted both `organization_id` and `organization_name` |
| `organization_not_found` | 404 | Super_admin passed an `organization_id` that doesn't exist |
| `org_slug_conflict` | 409 | The org name's slug collides with an existing, differently-named org |
| `org_name_produces_empty_slug` | 422 | Org name has no alphanumeric characters |
| `quota_exceeded` | 403 | Admin has hit their invite quota |
| `email_already_registered` | 409 | Duplicate email at invite time |
| `email_send_failed: ...` | 502 | Resend failed — invite/resend rolled back |
| `cannot_delete_self` / `cannot_resend_self` | 400 | Self-target on a destructive action |
| `cannot_delete_admin_or_super` / `cannot_resend_admin_or_super` | 403 | Admin targeted an admin/super_admin |
| `cross_org_forbidden` | 403 | Admin targeted a user outside their org |
| `invalid_role` | 400 | Role not in the locked 6-value enum |
| `user_not_found` | 404 | Target id doesn't exist |
| `org_not_activated` | 409/402 | Org action requires activation first (top-up), or the org can't spend yet (any metered feature) |
| `invalid_tier` | 422 | Tier not one of `free/pro/unlimited/payg` |

## 10. Common questions

**"I invited someone into a brand-new organization name and it 500'd."** —
This was a real bug (fixed 2026-07-27): `Organization.cycle_end` had no
default at the ORM level, so creating a fresh org without hand-specifying the
cycle window raised a not-null violation. Fixed by declaring the same
`server_default` the SQL DDL already had. If you see this again, it's a
regression of that fix.

**"An admin says invite is broken, but only sometimes."** — Historically,
invites into an *existing* organization worked fine while invites that had to
*create* the org 500'd (see above) — which is exactly the "seems broken"
pattern: it depends on whether the org already existed.

**"Why did deleting a user leave orphaned billing/profile rows?"** — The ORM
models don't declare foreign-key cascades (even though the real SQL schema
has them) — that's a known gap. The invite-rollback path works around it by
deleting the three rows explicitly rather than trusting a cascade; other
delete paths should do the same until the ORM/schema gap is closed.

**"Can an admin ever create another admin?"** — No, never — not for their
own org, not for any org. That's a hard rule enforced at role-resolution time
(`ADMIN_ASSIGNABLE_ROLES` never includes `admin`), independent of anything
else in the request.

## Things this page does not own

- JWT/session mechanics, middleware verification order, anonymous paths →
  [02-auth-and-rbac.md](02-auth-and-rbac.md).
- Exact column definitions for `users`, `users_profile`, `organizations` →
  [06-database-schema.md](06-database-schema.md).
- Endpoint-by-endpoint request/response schemas → [07-api-reference.md](07-api-reference.md).
- How the billing pool tied to each user/org actually works →
  [18-credits-and-billing-explained.md](18-credits-and-billing-explained.md).
- Project sharing / paid-tier project visibility rules →
  [05-storage-and-vectors.md](05-storage-and-vectors.md).
