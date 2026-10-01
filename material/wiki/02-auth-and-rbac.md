# 02 — Auth & RBAC

> _Owns: how identity, sessions, and role-based gates work end-to-end._
> _Changed 2026-06-07: auth is now self-hosted JWT (HS256) + bcrypt, not
> Supabase Auth. No JWKS, no `auth.users`, no Postgres `handle_new_user`
> trigger — the local `users` table is the source of truth._
> _Changed 2026-06-13: added a PUBLIC demo self-signup path (B2B stays
> invite-only). See [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md)
> for the demo lifecycle, token budget, and IP gating; this page owns the
> auth/credential mechanics (verification gate, anon paths, error codes)._

## Tiers

| Role | Origin | Scope |
|---|---|---|
| `super_admin` | Bootstrapped via CLI; can be cloned via the invite endpoint by another super_admin. | Unrestricted. Adds & deletes admins; sets per-admin quotas. |
| `admin` | Invited by a super_admin with `role: "admin"`, an `organization`, and a `user_quota`. | Org-scoped. Invites users in their own org only, up to `user_quota`. Cannot mint admins/super_admins. |
| `drilling_engineer` / `geologist` / `asset_manager` / `qa_steward` | Invited by an admin or super_admin. | End users. Can access their own projects/documents only. |

**B2B is invite-only** — every B2B account is created via
`POST /admin/users/invite`. There is **one** public path: the **demo
self-signup** (`POST /auth/demo/signup`), which creates an email-verified
`free`-plan demo user. Demo signups must confirm their email before they can
sign in (see below); invited users are created already-confirmed. The demo
budget, IP gating, and verification email are owned by
[14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

## How sign-in works

1. FE posts `{email, password}` to `/auth/login`.
2. Backend looks up the local `users` row and verifies the password with
   **bcrypt** ([app/auth/password.py](../app/auth/password.py)).
3. Backend mints a self-signed **HS256** access + refresh token (TTLs from
   `jwt_access_ttl_seconds=28800` / `jwt_refresh_ttl_seconds=86400`) and returns
   `{access_token, refresh_token, expires_in, token_type, user}`.
4. FE attaches `Authorization: Bearer <access_token>` to every request.
5. Before expiry the FE rotates via `/auth/refresh`. Both tokens are reissued.

## Middleware verification

[app/auth/middleware.py](../app/auth/middleware.py) does on every non-anonymous request:

1. Decode + verify the JWT against the local `JWT_SECRET` (**HS256**, no JWKS /
   `PyJWKClient`).
2. Load the user row by `id` to attach `role`, `display_name`, `organization`.
3. Set `request.state.user = AuthUser(...)`.
4. On any failure → `401`.

`OPTIONS` preflights are early-returned before auth so CORS works.

Anonymous paths (no JWT required):

```
/                          /openapi.json
/health                    /docs
/redoc                     /docs/oauth2-redirect
/auth/login                /auth/refresh
/auth/demo/signup          /auth/verify-email
/auth/demo/resend-verification
/welli/chat                /welli/chat/sync
/demo/request              /static/*  (prefix)
```

## Role enforcement

Two helpers in [app/auth/deps.py](../app/auth/deps.py):

- `current_user(request)` — `AuthUser` dataclass; raises 401 if absent.
- `require_role(*roles)` — Dependency factory. On mismatch returns
  `403 {error_code: "requires_role", required_roles, actual_role}`.

The admin-users router does not gate the whole router any more —
individual routes pick `require_role("super_admin")` or
`_require_inviter(...)` (admin or super_admin) so admins can call
invite/list/delete with org-scoping enforced inside the handler.

## Invite flow (one place to change)

[app/routes/admin_users.py](../app/routes/admin_users.py):

1. `_require_inviter(caller)` — must be admin or super_admin.
2. Resolve `requested_role`. Admin callers may not pick `admin` or
   `super_admin` → 403 `role_not_assignable_by_admin`.
3. Resolve `organization`:
   - Admin callers: forced to `caller.organization` (409 if null).
   - Super_admin: free choice via request body.
4. Quota check (admin only): count rows where `users_profile.invited_by = caller.id`.
   If `>= caller.user_quota` (and quota > 0) → 403 `quota_exceeded`.
5. Generate a `secrets.token_urlsafe(12)` password, bcrypt-hash it, and
   **insert the `users` + profile rows directly in one transaction** (no
   `auth.admin.create_user`, no `handle_new_user` trigger), setting `role`,
   `organization`, `user_quota`, `invited_by = caller.id`.
6. Send email via Resend ([app/auth/email.py](../app/auth/email.py)).
   On any send error: roll back / delete the row → 502 `email_send_failed`.

The resend-invite endpoint does steps 5b (admin update password) and 7
only — does not touch role/org/quota.

## Demo self-signup + email verification (one place to change)

[app/auth/routes.py](../app/auth/routes.py) (public; not the admin invite flow):

> _Changed 2026-06-14: demo signup is now **immediate-access / verify-in-background**
> (flag `demo_immediate_access`, default on). Signup returns tokens; verification
> is a soft gate._

1. `POST /auth/demo/signup` — per-IP signup cap (`enforce_demo_signup_quota`,
   429 on exceed) → reject if `email` already exists (409) → insert `users`
   (`email_confirmed=false`, `verification_token_hash = sha256(raw)`,
   `verification_sent_at=now`) + `users_profile` (`signup_source='demo_self_signup'`,
   default `drilling_engineer` role, no org) + `user_subscriptions`
   (`is_demo=true`, one-time budget, `signup_ip`) → email the link
   `{frontend_base_url}/verify-email?token=<raw>`. **Returns `201` with a token
   bundle (`status:"active_unverified"`, `email_verified:false`) — the user is
   signed in immediately.** Email-send failure doesn't roll back (user can resend).
2. `POST /auth/verify-email {token}` — looks up by `sha256(token)`, checks
   `demo_verification_ttl_seconds`, sets `email_confirmed=true`, clears the
   token, and returns a token bundle (`email_verified:true`).
3. `POST /auth/demo/resend-verification {email}` — per-IP capped; re-issues a
   token only for an unconfirmed account; **always** returns `202`.

**Login no longer hard-gates demo users**: with `demo_immediate_access` on,
`login()` allows `email_confirmed=false` (the `403 email_not_verified` only fires
if the flag is off). `AuthUser.email_verified` is loaded from `users.email_confirmed`
and surfaced on `/auth/me` + every token `user` payload. Verification is a **soft
gate** instead: unverified demo users get **no feedback bonus** and are **purged
after `demo_unverified_ttl_days`** (the stale-demo cleanup task /
`app.tools.purge_demo`). Only the raw token's SHA-256 hash is stored; tokens are
single-use. Budget/IP semantics → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

## Quota semantics

`user_quota` is meaningful **only for admins** (a number on every other
row is informational). 0 = unlimited.

The count uses `users_profile.invited_by = admin.id` rather than
"users in the same org" so that a super_admin manually adjusting an
admin's organization later doesn't retroactively change the quota count.

## Password change

`POST /auth/change-password` is the **only** way a user can change their
own password. Backend:

1. Verifies `current_password` against the stored bcrypt hash
   (`bcrypt.checkpw`).
2. Bcrypt-hashes `new_password` and updates the local `users` row.

A user signing in with a temp password can use the existing access
token to call this endpoint immediately; the token stays valid across
the password change.

## Bootstrapping super_admin

For a fresh environment (or a "no super_admin can sign in" emergency):

```powershell
.\.venv\Scripts\python.exe -m app.tools.bootstrap_super_admin `
    --email me@example.com `
    --password "yourStrongPassword" `
    --display-name "Your Name"
```

Idempotent: if the email exists, only the role is upgraded. See
[10-runbooks.md](10-runbooks.md).

## Error codes (FE-facing)

See [§0.0.8 of communication_from_backend.md](../frontend_docs/communication_from_backend.md)
for the full FE-facing list. Backend authoritative source:

- 401: `invalid_credentials`, `invalid_refresh_token`, `invalid_current_password`, `unauthorized`
- 403: `requires_role`, `role_not_assignable_by_admin`, `quota_exceeded`, `cross_org_forbidden`, `cannot_delete_admin_or_super`, `cannot_resend_admin_or_super`, `email_not_verified` (demo, until confirmed)
- 400: `cannot_delete_self`, `cannot_resend_self`, `invalid_role`, `password_update_failed: ...`, `invalid_or_used_token` / `verification_expired` (verify-email)
- 409: `email_already_registered`, `admin_missing_organization`
- 429: `demo_signup_rate_limited`, `demo_resend_rate_limited`, `demo_ip_limit`, `demo_request_rate_limited` (see [14](14-billing-demo-and-feedback.md))
- 502: `email_send_failed: ...`, `invite_no_user_returned`, `profile_not_created`

## Things this page does not own

- App-level ownership checks / vector tenant filter → [05-storage-and-vectors.md](05-storage-and-vectors.md).
- Schema definition of the `users` table → [06-database-schema.md](06-database-schema.md).
- Endpoint-by-endpoint contract → [07-api-reference.md](07-api-reference.md).
- Token budgets, credit gating, demo budget/IP limits, feedback → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).
