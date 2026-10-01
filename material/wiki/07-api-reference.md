# 07 — API Reference

> _Owns: every public HTTP endpoint and its request/response contract.
> Authoritative shape lives in [openapi.json](../openapi.json); this
> page is the human-readable summary._

## Conventions

- All non-anonymous endpoints require `Authorization: Bearer <jwt>`.
- All bodies are JSON unless noted (`/ingest/upload` is multipart).
- Identifiers in paths are uuids unless noted.
- Error shape (when not soft-landing): `{"detail": "<error_code or message>"}`
  or `{"detail": {"error_code": "...", ...}}` for structured errors.

## Anonymous

### `GET /health`

Returns `{"status": "ok"}` if the app is up. No auth.

### `POST /auth/login`

Body: `{ "email": str, "password": str }`

200: `{ access_token, refresh_token, expires_in, token_type: "bearer", user }`
where `user = { id, email, role, display_name, organization, email_verified }`.

401 `invalid_credentials` — wrong email or password.
429 `login_rate_limited` — per-IP / per-email login throttle (`Retry-After`).
403 `email_not_verified` — only when `demo_immediate_access` is **off** (a hard
gate). By default demo signups sign in unverified (soft gate); the `user.email_verified`
flag drives the "verify to keep your data" banner instead.

### `POST /auth/refresh`

Body: `{ "refresh_token": str }`

200: same shape as `/auth/login` (rotates both tokens).

401 `invalid_refresh_token`.

### `POST /auth/demo/signup` → 201

Public demo self-signup (B2B stays invite-only). Body:
`{ "email": str, "password": str (≥8), "display_name"?: str, "organization"?: str }`.

201 (default, `demo_immediate_access` on — **immediate access, verify in
background**): a full token bundle
`{ "status": "active_unverified", access_token, refresh_token, expires_in,
token_type, "email_verified": false, user }` — the user is signed in right away
and a verification email is also sent. With `demo_immediate_access` off it
returns `{ "status": "pending_verification", "email": str }` and no tokens.
409 `email_already_registered` · 429 `demo_signup_rate_limited`.
Behaviour → [14](14-billing-demo-and-feedback.md).

### `POST /auth/verify-email`

Body: `{ "token": str }`. 200: a token bundle (auto sign-in) like `/auth/login`.

400 `invalid_or_used_token` · 400 `verification_expired`.

### `POST /auth/demo/resend-verification` → 202

Body: `{ "email": str }`. Always 202 (never reveals whether the email exists or
is already verified). 429 `demo_resend_rate_limited`.

### `POST /demo/request`

Public "book a demo" lead. Body: `{ "name": str, "email": str, "company"?: str,
"message"?: str }`. 200: `{ "ok": true, "booking_url": str|null }`.
429 `demo_request_rate_limited`.

## Auth (bearer)

### `POST /auth/change-password` → 204

Body: `{ "current_password": str, "new_password": str }`. `new_password`
must be ≥ 8 chars.

401 `invalid_current_password` · 400 `password_update_failed: ...`.

### `GET /auth/me`

200: `{ id, email, role, display_name, organization, is_superadmin, email_verified }`.

### `PATCH /auth/me/profile`

Body (all optional): `{ "display_name": str?, "organization": str? }`.

200: refreshed `me` shape.

### `GET /me/usage`

Proactive token balance + allowance (powers the FE credits meter / Usage page).

200: `{ tier, unlimited, is_demo, balance, limit, bonus, cycle_end, caps,
org_consumed_credits?, org_threshold_credits?, org_alert_tier? }` —
`unlimited:true` ⇒ no numeric `limit` (super_admin / unlimited tier); free
(including demo signups) `limit` is the monthly grant (`free_monthly_token_budget`,
5M tokens); `caps` is `{}`. `balance`/`limit` stay in **tokens** for the FE meter
even though the ledger is credit-scaled (1 credit = 100,000 tokens, migration 018).
For paid-org members the informational odometer fields are populated:
`org_consumed_credits` (cumulative org spend this cycle, credits),
`org_threshold_credits` (per-org override or global default),
`org_alert_tier` (0–3, highest notification tier fired). Read-only; never gates.
See [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

## Admin / users (bearer; admin or super_admin)

### `GET /admin/users`

Query: `q`, `role`, `limit` (1-200, default 50), `offset` (≥0).

200: `{ items: UserRow[], total, limit, offset }` where

```
UserRow = { id, email, role, display_name, organization,
            user_quota, invited_by, created_at }
```

Admin sees only `caller.organization`. Super_admin sees everyone.

403 `requires_role` if caller is below admin tier.

### `POST /admin/users/invite` → 201

Body:
```json
{
  "email": "...",
  "display_name": "...",
  "role": "admin | super_admin | drilling_engineer | ...",
  "organization": "...",
  "user_quota": 5
}
```

Constraints:
- Admin caller: `role` must be in `{drilling_engineer, geologist, asset_manager, qa_steward}`. `organization` is ignored (forced to caller's).
- Super_admin: any role; `organization` is required when inviting an admin; `user_quota` only meaningful when inviting an `admin`.

201: `UserRow` + `message`.

403 `role_not_assignable_by_admin` · 403 `quota_exceeded` · 409 `email_already_registered` · 409 `admin_missing_organization` · 502 `email_send_failed: ...`.

### `PATCH /admin/users/{id}/role` → super_admin only

Body: `{ "role": str }`. Returns updated `UserRow`.

400 `invalid_role` · 404 `user_not_found`.

### `PATCH /admin/users/{id}/quota` → super_admin only

Body: `{ "user_quota": int }` (0–10000). Returns updated `UserRow`.

### `POST /admin/users/{id}/resend-invite`

Generates a new random password, bcrypt-hashes it into the user's row, mails it.

403 `cannot_resend_admin_or_super` (admin caller) · 403 `cross_org_forbidden` · 400 `cannot_resend_self` · 502 `email_send_failed: ...`.

### `DELETE /admin/users/{id}` → 204

Admin can delete only same-org non-admin users. Cannot delete self.

403 `cannot_delete_admin_or_super` · 403 `cross_org_forbidden` · 400 `cannot_delete_self`.

## Admin / organizations (bearer; super_admin only)

### `GET /admin/organizations`

200: `{ items: OrgRow[], total, limit, offset }` where

```
OrgRow = { id, name, slug, tier, status, activated_at, overage_allowed,
           credit_balance, bonus_credits, consumed_credits_cycle,
           alert_tier_fired, alert_threshold_credits, alerts_enabled,
           created_at }
```

Balances are on the **credit** scale (1 credit = 100,000 tokens, migration 018).

### `PATCH /admin/organizations/{id}` → super_admin only

Body (all optional): `{ name?, tier?, overage_allowed?, alert_threshold_credits?,
alerts_enabled? }`. Slug is frozen. `tier` ∈ `{free, pro, unlimited, payg}`
(else 422 `invalid_tier`).

- `alert_threshold_credits` (`float > 0`): per-org spend-alert threshold in
  credits (`null`/omitted keeps the current value; there is no clear-to-default).
- `alerts_enabled` (`bool`): per-org kill switch for the spend-alert odometer.

The last two drive the **notification-only** org spend-alert (1×/2×/3× the
threshold, admins-only, never blocks usage). Returns the updated `OrgRow`.
404 `organization_not_found`. See
[14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

> Other org lifecycle endpoints exist under `/admin/organizations` (`POST` to
> create, `POST /{id}/activate`, `POST /{id}/top-up`) — see the route module
> [app/routes/admin_organizations.py](../app/routes/admin_organizations.py).

## Admin / projects (bearer; admin or super_admin)

Org-project governance ([app/routes/admin_projects.py](../app/routes/admin_projects.py), prefix `/admin/projects`).

| Method | Path | Notes |
|---|---|---|
| `POST` | `/admin/projects/{id}/share` | Make a project org-wide shared (project must have an org → 409 `project_has_no_org`) |
| `POST` | `/admin/projects/{id}/unshare` | Make it personal again |
| `POST` | `/admin/projects/{id}/transfer` | Transfer ownership to another same-org user |
| `POST` | `/admin/projects/{id}/members` | Grant a user access to the project |
| `GET` | `/admin/projects/{id}/members` | List project members |
| `DELETE` | `/admin/projects/{id}/members/{uid}` | Revoke a member's access |

## Admin / usage (bearer; admin or super_admin)

### `GET /admin/usage`

Aggregated token/cost usage rollup ([app/routes/admin_usage.py](../app/routes/admin_usage.py)).
Admin sees own org; super_admin sees all (`?scope=all`). See
[14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

## Admin / credits (bearer; admin or super_admin, except where noted)

Credit + tier administration ([app/routes/admin_credits.py](../app/routes/admin_credits.py), prefix `/admin`).

| Method | Path | Notes |
|---|---|---|
| `PATCH` | `/admin/users/{uid}/subscription` | Set a user's tier |
| `PATCH` | `/admin/orgs/{oid}/subscription` | Set an org's pool tier (super_admin) |
| `POST` | `/admin/users/{uid}/credits` | Grant bonus credits |
| `GET` | `/admin/users/{uid}/credits` | A user's live balance |
| `GET` | `/admin/plans` | List subscription plans (super_admin) |
| `PUT` | `/admin/plans/{key}` | Edit a plan's allowances/caps (super_admin) |

Balances are credit-scaled (1 credit = 100,000 tokens, migration 018).

## Projects (bearer)

| Method | Path | Notes |
|---|---|---|
| `POST` | `/projects` | Body `{ name, description? }` → `{ id, ... }` |
| `GET` | `/projects` | `{ count, projects: [...] }` |
| `GET` | `/projects/{id}` | Project detail |
| `PATCH` | `/projects/{id}` | Body `{ name?, description? }` |
| `DELETE` | `/projects/{id}` | Cascades documents/jobs/Storage/Qdrant |

## Documents (bearer)

> Text/JSON responses are gzip-compressed (GZipMiddleware). Image/PDF
> endpoints return bytes directly (no Supabase signed URLs). See
> [13-document-viewer-and-serving.md](13-document-viewer-and-serving.md) for the
> lazy-loading viewer contract.

| Method | Path | Notes |
|---|---|---|
| `GET` | `/documents` | `{ count, documents: [...] }`. Optional `?project_id=`; `?scope=all` (super_admin) |
| `GET` | `/documents/{id}` | Metadata + `latest_job` |
| `PATCH` | `/documents/{id}` | Governance: `{ artifact_class?, confidentiality?, superseded_by? }` |
| `POST` | `/documents/{id}/auto-classify` | Re-run LLM artifact-class classifier (`?force=`) |
| `DELETE` | `/documents/{id}` | Cascades local storage + Qdrant + hash cache |
| `GET` | `/documents/{id}/full` | Full markdown (`text/plain`). Legacy whole-doc blob |
| `GET` | `/documents/{id}/manifest` | `{ page_count, ingest_status, pages_done, headings[] }` — lightweight, for virtualized viewers |
| `GET` | `/documents/{id}/pages/{n}/markdown` | One page's markdown (lazy per-page reader) |
| `GET` | `/documents/{id}/file` | Source PDF, range-capable (`Accept-Ranges`/`206`) for pdf.js |
| `GET` | `/documents/{id}/pages/{n}/image` | Rendered PNG/WebP **bytes** (200). `?w=160\|320\|800\|1600`, `?fmt=webp` |
| `GET` | `/documents/{id}/pages/{n}/image/crop` | Bbox crop. Query: `ymin, xmin, ymax, xmax` (0..1000) |

## Ingest (bearer)

| Method | Path | Notes |
|---|---|---|
| `POST` | `/ingest` | Body `{ project_id, file_paths: [...] }` → `[{ job_id, document_id, filename, status }]` |
| `POST` | `/ingest/upload` | Multipart: `project_id` + `files`. Same response. |
| `GET` | `/ingest` | `{ count, jobs: [...] }` scoped to caller |
| `GET` | `/ingest/{job_id}` | Single job row (404 if not yours) |

## Search (bearer)

`GET /search`

Query (required): `q`. Optional: `limit` (1-100, default 10), `project_id`, `document_id`, `page_num`, `score_threshold`.

200: list of chunks with `{ score, text, document_id, page_num, ... }`.

## Insights — NPT + Lessons (bearer)

Structured NPT + Lessons facts extracted at ingest. ACL like `/search`; the
GETs settle ~0 credits, RCA is metered. Contracts owned by
[Well Knowledge Layer](17-well-knowledge-layer.md) and mirrored for the FE in
[communication_from_backend.md §0.21](../frontend_docs/communication_from_backend.md).

- `GET /projects/{project_id}/insights` — cached top-5 NPT (by hours) + top-5 Lessons for one project. `state`: `processed`/`empty`/`pending`.
- `GET /projects/insights?breakdown=by_project` — global rank across all accessible projects; optional per-project breakdown.
- `GET /documents/{document_id}/insights` — raw per-document NPT + Lessons (Layer A). Never 404s on state.
- `POST /documents/{document_id}/insights/reextract` — re-run extraction → projection → rollup for one document, reusing its stored markdown (no PDF re-ingest). Metered.
- `POST /projects/{project_id}/rca` — root-cause analysis; body `{ "symptom": "…" }` (optional). Returns deterministic `evidence` + markdown `analysis` with citations.

`npt_events.category` is a CLOSED vocabulary (safe to `GROUP BY`); `category_raw` holds the report's original wording. Rows carry `document_name` for human-readable citations. Wells are reserved (`wells: []`) until well-identity resolution ships.

## Chat (bearer)

### `POST /chat`

Body:
```json
{ "message": "...", "session_id": "...?", "project_id": "...?",
  "document_id": "...?", "temp": false }
```

`session_id` threads the turn into an existing conversation (omit to start a new
one; the response echoes the id). `temp: true` is an incognito turn — no
long-term memory capture, 1-hour Redis session TTL.

200: `ChatResponse` — `{ answer, sources[], evidence[], claims[], unanswered[],
attached_pages[], session_id, meta }`. `sources[]` carry `source_id, text, score,
doc_name, page_num, kind, collection, ...`; `evidence[]` are render cards
(`kind: table|bbox_crop|page_image`). See [04](04-retrieval-and-chat.md).

### `POST /chat/stream`

Same body as `POST /chat`. Returns an **SSE** stream (progressive answer + a
final `done` frame carrying `meta`, `sources`, `evidence`). See
[04](04-retrieval-and-chat.md) for the frame protocol.

## Chat sessions (bearer)

Persisted conversation log ([app/routes/chat_sessions.py](../app/routes/chat_sessions.py), prefix `/chat/sessions`).

| Method | Path | Notes |
|---|---|---|
| `GET` | `/chat/sessions` | List the caller's sessions (newest first) |
| `PATCH` | `/chat/sessions/{id}` | Rename a session |
| `DELETE` | `/chat/sessions/{id}` | Delete a session |
| `GET` | `/chat/sessions/{id}/messages` | Hydrate the persisted message log (`?limit`, ≤1000) |
| `POST` | `/chat/sessions/{id}/archive` | Hide from the default sidebar |
| `POST` | `/chat/sessions/{id}/unarchive` | Restore an archived session |

## Memory (bearer)

Cross-session (long-term) memory ([app/routes/memories.py](../app/routes/memories.py), prefix `/memories`). See [11](11-memory-and-chatgpt-roadmap.md).

| Method | Path | Notes |
|---|---|---|
| `GET` | `/memories/settings` | Get the caller's memory toggle |
| `PATCH` | `/memories/settings` | Toggle cross-session memory on/off |
| `GET` | `/memories` | List the caller's stored memories |
| `DELETE` | `/memories/{scope}/{key}` | Forget a single memory |
| `POST` | `/memories/forget-all` | Forget every memory for the caller |

## Reports (bearer)

Strategic report generation (see [15-report-generation.md](15-report-generation.md)).
Router prefix `/projects/{project_id}/reports`.

| Method | Path | Notes |
|---|---|---|
| `POST` | `/projects/{id}/reports` | Body `{ title?, template?="well_strategic_v1", options?: {coverage:"auto\|per_document\|project"} }` → **202** `{ report_id, status:"queued" }`. One-in-flight per project (409 if a run is active). Reserves credits. |
| `GET` | `/projects/{id}/reports` | List the project's reports (status + timing). |
| `GET` | `/projects/{id}/reports/{report_id}` | Report record: `status` (`queued\|running\|succeeded\|failed`), `current_phase`, `sections_total/done`, `markdown` (when succeeded), `sections_meta`, and — agentic path only — `blackboard` + `quality_score` (both null on legacy). Poll this. |
| `GET` | `/projects/{id}/reports/{report_id}/pdf` | Rendered PDF (StreamingResponse). Embedded figures are inlined server-side so they appear in the download. |
| `GET` | `/projects/{id}/reports/{report_id}/docx` | Rendered Word `.docx` (StreamingResponse), same content + inlined figures as the PDF. |
| `DELETE` | `/projects/{id}/reports/{report_id}` | Delete a report → **204**. Owner-scoped (404 if not yours); works for any status (incl. a stuck queued/running one). |

Generation runs async (Celery); poll the GET. The **agentic path**
(`report_agentic_enabled`) is API-compatible but produces richer output and takes
materially longer — poll patiently. Metered: a `402` (budget) or `429`
(`demo_ip_limit`) can be returned on POST (see Credit gating below).

## Feedback (bearer)

### `POST /feedback`

Body: `{ "rating": int (1–5), "message": str }` (`message` ≥ `feedback_min_chars`).

200: `{ feedback_id, bonus_granted, bonus_tokens, grants_remaining, bonus_balance }`.
A qualifying submission grants bonus tokens **once per user lifetime** and emails
the team. 400 `feedback_too_short`. Behaviour → [14](14-billing-demo-and-feedback.md).

## Support (bearer)

User-facing support tickets ([app/routes/support.py](../app/routes/support.py), prefix `/support`).

| Method | Path | Notes |
|---|---|---|
| `POST` | `/support/tickets` | **Multipart** submit: `category`, `subject`, `message`, `include_diagnostics?`, `diagnostics?`, `attachments?`. Per-user hour/day throttle (429 `rate_limited`) |
| `GET` | `/support/tickets/me` | My tickets (newest first, cursor-paginated; `?status`, `?cursor`, `?limit`) |
| `GET` | `/support/tickets/me/{id}` | Single ticket detail (owner only) |
| `GET` | `/support/tickets/{id}/attachments/{aid}` | Fetch an attachment (redirects to a served URL) |

## Admin / support (bearer; admin or super_admin)

Ticket triage ([app/routes/admin_support.py](../app/routes/admin_support.py), prefix `/admin/support`).

| Method | Path | Notes |
|---|---|---|
| `GET` | `/admin/support/tickets` | List all tickets (triage) |
| `GET` | `/admin/support/tickets/{id}` | Single ticket detail (admin view) |
| `PATCH` | `/admin/support/tickets/{id}` | Update status / add an admin note |
| `DELETE` | `/admin/support/tickets/{id}` | Soft-delete a ticket |

## Report templates (bearer)

Reusable report templates + ad-hoc generation ([app/routes/report_templates.py](../app/routes/report_templates.py), prefix `/report-templates`). See [15](15-report-generation.md).

| Method | Path | Notes |
|---|---|---|
| `GET` | `/report-templates` | List templates (built-in + custom) |
| `GET` | `/report-templates/{id}` | Template detail |
| `POST` | `/report-templates` → 201 | Create a custom template |
| `PATCH` | `/report-templates/{id}` | Update a custom template |
| `DELETE` | `/report-templates/{id}` → 204 | Delete a custom template |
| `POST` | `/report-templates/generate` | Ad-hoc generate from a template |

## Welli (anonymous)

Public marketing-site assistant ([app/routes/welli.py](../app/routes/welli.py), prefix `/welli`). No auth.

| Method | Path | Notes |
|---|---|---|
| `POST` | `/welli/chat` | Chat with Welli (**SSE** stream) |
| `POST` | `/welli/chat/sync` | Chat with Welli (JSON response) |

## Credit gating (chat · search · ingest · report)

These metered endpoints **reserve** a token estimate up front and **settle**
against measured usage. When the caller is out of budget and their tier is
enforced (free always when `free_tier_enforced`; paid via
`billing_enforcement_enabled`):

`402` `{ error_code: "insufficient_credits", balance, required, upgrade_hint,
earn_more_hint: "feedback"?, book_demo_hint: bool }`.

Demo users are additionally IP-gated → `429 { error_code: "demo_ip_limit",
book_demo: true }`. Full model → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).

## Smoke coverage

[app/tools/api_smoke.py](../app/tools/api_smoke.py) exercises 23 of
the above contracts in ~30 seconds against a live stack. See
[09-testing.md](09-testing.md).
