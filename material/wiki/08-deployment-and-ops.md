# 08 — Deployment & Ops

> _Changed 2026-06-07: updated for the Supabase→local Docker stack. Owns:
> how to run the stack, required env, scaling knobs, log channels._
> _Changed 2026-06-25: centralized structlog request/pipeline logging under
> `app.observability` and added optional Azure Application Insights export via
> OpenTelemetry (`APPLICATIONINSIGHTS_CONNECTION_STRING`)._

## Local stack (docker-compose)

```powershell
docker compose up -d
docker compose ps
docker logs -f wellsynthai-app
```

Services (all in-stack): `wellsynthai-postgres`, `wellsynthai-qdrant`,
`wellsynthai-redis`, `wellsynthai-app`, `wellsynthai-celery-worker`,
`wellsynthai-celery-batch-worker`. They start in dependency order — `postgres`,
`qdrant`, `redis` are healthchecked before `app`/workers start. API health is
`GET /health`.

Source is **live-mounted** (`./app:/app/app`), but the server loads modules at
startup (no `--reload`). Code changes need a restart:

```powershell
docker compose restart app celery-worker celery-batch-worker
```

Horizontal scale-out: `docker compose up -d --scale celery-worker=N`.

## Required env (.env)

Copy [.env.example](../.env.example) to `.env`. There are **no Supabase vars**.

| Var | Purpose |
|---|---|
| `POSTGRES_PASSWORD` | Local Postgres password (default `wellsynthai_dev`) |
| `DATABASE_URL` | `postgresql+asyncpg://wellsynthai:<pw>@postgres:5432/wellsynthai` |
| `WORKER_DATABASE_URL` | Same as `DATABASE_URL` (workers create/dispose engine per task) |
| `CELERY_BROKER_URL` | `redis://redis:6379/0` |
| `CELERY_RESULT_BACKEND` | `redis://redis:6379/1` |
| `REDIS_URL` | `redis://redis:6379/2` (app/session state) |
| `QDRANT_URL` | `http://qdrant:6333` |
| `JWT_SECRET` | HS256 signing key (≥ 32 chars) — **required for auth** |
| `GEMINI_API_KEY` | Extraction + chat + embeddings |
| `STORAGE_ROOT` / `UPLOAD_DIR` | Local artifact + upload dirs (`./data/storage`, `./data/uploads`) |
| `RESEND_API_KEY` / `RESEND_FROM_EMAIL` / `RESEND_FROM_NAME` | Outbound invite email (optional) |

Optional: `LOG_LEVEL` (default `INFO`), `CORS_ALLOW_ORIGINS` (default
`http://localhost:3000`; `*` = wide open).

Observability env:

| Var | Purpose |
|---|---|
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Azure Application Insights connection string; blank disables Azure export |
| `TELEMETRY_ENABLED` | Master switch for Azure/OpenTelemetry setup (`true` by default) |
| `TELEMETRY_SERVICE_NAME` | Service name sent to Azure Monitor (`wellsynthai-backend`) |
| `TELEMETRY_ENVIRONMENT` | Deployment environment tag (`local`, `staging`, `prod`) |
| `TELEMETRY_RESOURCE_DETECTORS` | Azure resource detectors; default `azure_app_service` avoids Azure VM IMDS probes in containers |
| `TELEMETRY_STATSBEAT_ENABLED` | Exporter self-telemetry; default `false` to avoid repeated `169.254.169.254` metadata probes outside Azure VM |
| `REQUEST_LOGGING_ENABLED` | Emits one structured completion log per HTTP request |
| `REQUEST_LOG_EXCLUDED_PATHS` | Comma-separated paths excluded from request logs (default `/health`) |

## First-time bootstrap

1. `docker compose up -d` — `migrations/init.sql` runs via the Postgres initdb
   hook on a fresh volume, then the one-shot **`migrate`** service applies pending
   migrations with `alembic upgrade head` *before* app/workers start
   (`depends_on: service_completed_successfully`). Manual `psql` apply is only a
   recovery path — see [10-runbooks.md](10-runbooks.md).
2. Bootstrap a super_admin (CLI in [10-runbooks.md](10-runbooks.md)).
3. Verify the Resend domain (if sending invites).
4. Smoke: `python -m app.tools.api_smoke` against a live stack.

## Scaling / safety knobs

| Knob | Default | Where |
|---|---|---|
| Per-process Gemini in-flight cap | `max_concurrent_pages=60` | [app/config.py](../app/config.py) |
| Celery realtime concurrency | `--concurrency=4` | `docker-compose.yml` (celery-worker) |
| Celery batch concurrency | `--concurrency=2` | `docker-compose.yml` (celery-batch-worker) |
| Worker recycle | `--max-tasks-per-child=5`, `--prefetch-multiplier=1` | `docker-compose.yml` |
| Uvicorn workers | 2 | `docker-compose.yml` (app CMD) |
| Render DPI / threads | `render_dpi=200`, `render_concurrency=8` | [app/config.py](../app/config.py) |
| Table-route gate | `route_table_min_rows/cols=2` | [app/config.py](../app/config.py) |
| Max upload | `max_upload_mb=500` | [app/config.py](../app/config.py) |
| Mem limits | app 2g · worker 8g · batch-worker 6g | `docker-compose.yml` |
| Agentic reports | `report_agentic_enabled=False` (+ `report_master/planner/grader_model`, `report_section_max_reflections=2`) | [app/config.py](../app/config.py); see [15](15-report-generation.md) |
| Report section concurrency | `report_max_concurrent_sections_per_worker=10` | [app/config.py](../app/config.py) |
| Free-tier / demo gating | `free_tier_enforced`, `demo_*`, `report_web_search_enabled` | [app/config.py](../app/config.py); see [14](14-billing-demo-and-feedback.md) |

> Turning on `report_agentic_enabled` raises report cost & latency materially
> (top-tier master + N per-section agents + reflection). Roll out per-environment.
> Needs migration `008_report_blackboard.sql` (see [10-runbooks.md](10-runbooks.md)).

> Memory note: one document = one prefork child; `--concurrency=4` large docs in
> one 8g container can OOM. See the worker-distribution memory analysis and
> [PLAN_pdf_sharding.md](../data/plan_documentations/PLAN_pdf_sharding.md).

## Logs & observability
- App: `docker logs wellsynthai-app`; Worker: `wellsynthai-celery-worker`;
  Batch: `wellsynthai-celery-batch-worker`; plus `postgres`, `qdrant`, `redis`.
- Centralized code lives in [app/observability](../app/observability). Existing
  imports use [app/utils/logging.py](../app/utils/logging.py), which re-exports
  the centralized logger helpers for compatibility.
- App logs remain structured stdout (`LOG_FORMAT=json|console`). Every HTTP
  request gets a `request_id` and an `http_request_completed` event unless the
  path is excluded.
- Azure Application Insights export is optional: set
  `APPLICATIONINSIGHTS_CONNECTION_STRING` and install the pinned requirements.
  If the env var or Azure package is missing, the app logs a telemetry-disabled
  event and continues locally.
- structlog key=value lines. Grep keys:
```
http_request_completed, http_request_failed,
pipeline_complete, pipeline_failed, pipeline_degraded_pages_high,
worker_page_failed, gemini_call_error, gemini_call_empty_text,
chat_agent_error, resend_send_failed, invite_email_sent
```

## Hot paths to monitor
- `gemini_call_error` bursts with `error_type=ConnectError/ReadError` and
  `rate_limited=False` → connection-storm at job start; check pacing/timeouts.
- `pipeline_degraded_pages_high` → many pages fell back to baseline text.
- `resend_send_failed` → from-domain unverified or quota exhausted.

## Things this page does not own
- Migration mechanics → [10-runbooks.md](10-runbooks.md).
- Test commands → [09-testing.md](09-testing.md).
- TLS certificates, nginx HTTPS and prod VM rollout → [22-tls-and-https.md](22-tls-and-https.md).
