# 10 — Runbooks

> _Owns: copy-pasteable operational procedures. Every runbook here
> is something we have actually executed in production or staging._

All commands assume you are at the repo root with the venv active:

```powershell
cd d:\Sriharsha\professional\Wellsynthai
.\.venv\Scripts\Activate.ps1
```

## Bootstrap a super_admin

```powershell
.\.venv\Scripts\python.exe -m app.tools.bootstrap_super_admin `
    --email super-admin@data-smith.ai `
    --password 12345@st67890 `
    --display-name "Super Admin"
```

Idempotent: promotes the email if it already exists, otherwise creates the
`users` row (bcrypt-hashed password) + profile directly and sets the role to
`super_admin`. Prints the user id on success. (No `handle_new_user` trigger —
that was Supabase; rows are inserted by app code now.)

## Add a second super_admin

Three equivalent options:

1. **UI** — sign in as super_admin → Super Admin Console → "Add super
   admin" tab. (Requires FE deployed.)
2. **API** — `POST /admin/users/invite` with `role: "super_admin"`.
   See [07-api-reference.md](07-api-reference.md).
3. **CLI** — re-run `bootstrap_super_admin.py` with the new email.

## Recover a lost password

A user reports they cannot log in:

```powershell
# As super_admin, get the user id
curl -H "Authorization: Bearer $TOKEN" "http://localhost:8080/admin/users?q=lost@user.com"

# Trigger a fresh-password email
curl -X POST `
  -H "Authorization: Bearer $TOKEN" `
  "http://localhost:8080/admin/users/<user_id>/resend-invite"
```

The endpoint generates a new random password, bcrypt-hashes it into the local
`users` row, and emails it via Resend. The previous password is invalidated.

## Apply migrations

Migrations live in [migrations/](../migrations/): `init.sql` (consolidated base
schema) + numbered increments `002_batch_lane` · `003_usage_events` ·
`004_organizations_shared_projects` · `005_credit_subscriptions` ·
`006_free_tier_tokens_feedback` · `007_demo_signup` · `008_report_blackboard` ·
`009_documents_org_id` · `010_rescale_token_balances` · `011_demo_is_free` ·
`012_credit_holds` · `013_usage_observability` · `014_org_mandatory` ·
`015_project_members` · `016_payg_activation` · `017_super_admin_org` ·
`018_credit_redenomination`.
Schema changes use handwritten SQL migrations as the deploy authority, with
SQLAlchemy models as the application mapping and drift-check surface. All files
are idempotent (`IF NOT EXISTS` / `ON CONFLICT DO NOTHING`).

- **Fresh DB**: `init.sql` is *supposed* to run via the Postgres initdb hook
  (`/docker-entrypoint-initdb.d/init.sql`) — but the hook **only fires on a brand-new
  data volume**. ⚠️ If the volume predates the mount, init never ran and the DB is
  empty (`\dt` shows nothing, app errors with `relation "public.X" does not exist`).
- **Connect** (service name is `postgres`, not `wellsynthai-postgres`):
  ```powershell
  docker compose exec -T postgres psql -v ON_ERROR_STOP=1 -U wellsynthai -d wellsynthai < migrations/<file>.sql
  ```

### Automated runner (preferred — how prod stays in sync)

Migrations are **baked into the app image** (`Dockerfile COPY . .` → `/app`)
and applied by Alembic — the enterprise-standard migration runner used with
SQLAlchemy. The first Alembic revision adopts the existing numbered SQL files;
future schema changes should be normal Alembic revisions under
`alembic/versions/`.

```powershell
# Apply (run from any host with the image; uses DATABASE_URL from env/settings)
docker compose run --rm migrate                       # the one-shot service
# or directly:
docker compose run --rm app alembic upgrade head
docker compose run --rm app alembic current
docker compose run --rm app alembic history
```

The prod `docker-compose.yml` defines a one-shot **`migrate`** service that runs
`alembic upgrade head` BEFORE `app`/`celery-*` start (`depends_on: condition:
service_completed_successfully`), gated on a Postgres healthcheck — so a normal
`docker compose up -d` applies pending migrations first, then boots the app.

**Deploying a new migration to prod:**
1. Update [app/db/models.py](../app/db/models.py) if the ORM shape changes.
2. Create an Alembic revision: `alembic revision -m "short description"`.
3. Put explicit `upgrade()` DDL in the revision. Add `downgrade()` when safe.
4. For fresh-volume parity, mirror the final schema in `migrations/init.sql` until we remove the initdb baseline.
5. Run the ORM coverage check: `docker compose run --rm app python -m app.tools.check_schema_parity`.
6. Build & push the backend image (Alembic revisions are baked in).
7. On the VM: `docker compose pull && docker compose up -d` → `migrate` applies it.
   (Back up first: `docker compose exec -T postgres pg_dump -U wellsynthai wellsynthai > backup.sql`.)

The legacy [app/tools/apply_migrations.py](../app/tools/apply_migrations.py)
is kept only as a fallback for the pre-Alembic numbered SQL files. Do not add new
schema changes there.

### Manual clean apply (or recover an empty DB)
Apply the base, then every increment in order. Idempotent + `ON_ERROR_STOP=1`:
```powershell
docker compose exec -T postgres psql -v ON_ERROR_STOP=1 -U wellsynthai -d wellsynthai < migrations/init.sql
foreach ($f in '002_batch_lane','003_usage_events','004_organizations_shared_projects','005_credit_subscriptions','006_free_tier_tokens_feedback','007_demo_signup','008_report_blackboard','009_documents_org_id','010_rescale_token_balances','011_demo_is_free','012_credit_holds','013_usage_observability','014_org_mandatory','015_project_members','016_payg_activation','017_super_admin_org','018_credit_redenomination') {
  docker compose exec -T postgres psql -v ON_ERROR_STOP=1 -U wellsynthai -d wellsynthai < "migrations/$f.sql"
}
```
`NOTICE ... already exists, skipping` lines are expected (init.sql already created
most objects). Verify, then **restart so the super_admin bootstrap can seed**:
```powershell
docker compose exec -T postgres psql -U wellsynthai -d wellsynthai -tAc "SELECT count(*) FROM pg_tables WHERE schemaname='public';"
docker compose restart app celery-worker celery-batch-worker
```

To add a migration:
1. Create `migrations/00N_<short_description>.sql`, idempotent (`IF NOT EXISTS`).
2. Mirror the change in [app/db/models.py](../app/db/models.py) **and** `migrations/init.sql`.
3. Apply it with the runner or `psql` command above.
4. Run `python -m app.tools.check_schema_parity` against the target database.

## Reset Qdrant (full wipe)

> Destructive. Requires re-ingesting every document.

```powershell
.\.venv\Scripts\python.exe -m app.tools.qdrant_reset
```

Calls `store.reset_all()` — drops and recreates the org-scoped collection set
(default: `wellsynthai_documents`, `wellsynthai_entities`, `wellsynthai_images`)
with the correct 768-d vectors + payload indices.

## Replay an ingest

If a job is `failed` and you've fixed the underlying bug:

```powershell
# Find the document_id from the failed job
curl -H "Authorization: Bearer $TOKEN" "http://localhost:8080/ingest" | jq

# Re-ingest by uploading the same file again.
# The document row is reused (matched on user_id + project_id + filename)
# so the document_id stays stable.
```

We do not currently support "rerun this job id" — the contract is
"re-upload the file"; the document identity persists.

## Verify Resend domain

If invite emails 502 with `email_send_failed`:

1. Log into Resend dashboard → Domains.
2. Confirm `wellsynth.ai` shows "Verified".
3. If not, re-add the SPF/DKIM records in DNS.
4. Test with `curl -X POST .../admin/users/invite ...` to your own inbox.

## Promote / demote a user role

```powershell
curl -X PATCH `
  -H "Authorization: Bearer $TOKEN" `
  -H "Content-Type: application/json" `
  -d '{"role": "admin"}' `
  "http://localhost:8080/admin/users/<user_id>/role"
```

Allowed values: `super_admin`, `admin`, `drilling_engineer`, `geologist`,
`asset_manager`, `qa_steward`. Super_admin only.

## Set / change an admin's user_quota

```powershell
curl -X PATCH `
  -H "Authorization: Bearer $TOKEN" `
  -H "Content-Type: application/json" `
  -d '{"user_quota": 25}' `
  "http://localhost:8080/admin/users/<admin_id>/quota"
```

`0` means unlimited. Quota is checked at invite time against the count
of profiles where `invited_by = admin.id`.

## Restart the live stack after env / code change

```powershell
docker compose restart app celery-worker celery-batch-worker
```

Usually enough (the server loads modules at startup — no `--reload`). Full
rebuild (only if Dockerfile / deps changed):

```powershell
docker compose build
docker compose up -d
```

## Smoke after any deploy

```powershell
.\.venv\Scripts\python.exe -m app.tools.api_smoke
```

Expect `failed: 0`. If any step fails, do **not** declare the deploy good —
fix forward or roll back.

## Things this page does not own

- Why a given step is needed → the topical wiki page.
- New runbooks for features that don't exist yet — add them here only
  after you've executed the procedure for real.
