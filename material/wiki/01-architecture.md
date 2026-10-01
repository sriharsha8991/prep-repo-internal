# 01 — Architecture

> _Changed 2026-06-07: updated for the Supabase→local migration. Owns:
> service topology, where every long-lived process lives, request
> lifecycle, links to deeper pages for each subsystem._

## Services (docker-compose)

| Service | Container | Purpose |
|---|---|---|
| API | `wellsynthai-app` | FastAPI on `:8080` (2 uvicorn workers). `mem_limit 2g`. |
| Worker | `wellsynthai-celery-worker` | Celery prefork pool, `--concurrency=4`. Queues: `extract`, `rasterize`, `aggregate`, `reports`, `default`. `mem_limit 8g`. |
| Batch worker | `wellsynthai-celery-batch-worker` | Dedicated to `batch_poll`, `batch_collect` (Gemini Batch lane), `--concurrency=2`. `mem_limit 6g`. |
| Postgres | `wellsynthai-postgres` | `postgres:16-alpine` — system of record (users, projects, documents, jobs, usage, reports, support, memories). |
| Vectors | `wellsynthai-qdrant` | Org-scoped collections; tenant-scoped via payload filters. See [05](05-storage-and-vectors.md). |
| Broker | `wellsynthai-redis` | Celery broker (`/0`) + result backend (`/1`) + app/session state (`/2`). |

Everything runs in-stack now. The only **external** dependencies are:
- **Gemini** — `gemini-2.5-flash-lite` (extraction/classify/lite chat) and
  `gemini-3.5-flash` (escalation/synth); `gemini-embedding-001` (768-d).
- **Resend** — transactional email (invite + resend-invite).

Storage is the **local filesystem** (`./data/storage`, mounted into app +
workers), not Supabase Storage. Auth is **self-hosted JWT**, not Supabase Auth.

## Request lifecycle

```
HTTP → FastAPI → AuthMiddleware → CORSMiddleware → GZipMiddleware → router → handler
                                                                       ↓
                                                  Postgres / Qdrant / local FS / Celery
```

Middleware (outermost → innermost, from [app/main.py](../app/main.py)):
- `AuthMiddleware` — verifies a self-signed **HS256** JWT against `JWT_SECRET`
  (no JWKS), loads the user row, sets `request.state.user`. `OPTIONS` and
  anonymous paths bypass. Anonymous: `/`, `/health`, `/openapi.json`, `/docs`,
  `/redoc`, `/docs/oauth2-redirect`, `/auth/login`, `/auth/refresh`,
  `/welli/chat`, `/welli/chat/sync`, `/static/*`.
- `CORSMiddleware` — env-driven allow-list (`CORS_ALLOW_ORIGINS`), default
  `http://localhost:3000`.
- `GZipMiddleware` — compresses text/JSON responses ≥ 500 bytes.

## Routers  ·  [app/main.py](../app/main.py)
`auth`, `admin_users`, `admin_usage`, `projects`, `documents`, `ingest`,
`search`, `chat`, `chat_sessions`, `memories`, `support`, `admin_support`,
`reports`, `report_templates`, `welli`.

## Background work (Celery)
- `POST /ingest` writes a `Job` row, stores the PDF, dispatches `ingest_document`
  (realtime) or `ingest_batch_submit` (batch lane).
- The orchestrator runs page-streamed extraction (60 concurrent asyncio workers
  per doc). See [03-ingestion-pipeline.md](03-ingestion-pipeline.md).
- Status is written to the Postgres `jobs` row; the FE **polls**
  `GET /ingest/{job_id}` (Supabase Realtime was removed in the migration).

## Module map (1-line each)
```
app/
  main.py             — FastAPI app, middleware (Auth/CORS/GZip), router include, lifespan singletons
  config.py           — Settings (pydantic-settings) from env
  auth/               — HS256 JWT verify, AuthMiddleware, /auth routes, deps, bcrypt password, email
  routes/             — admin_users, admin_usage, projects, documents, ingest, search, chat, chat_sessions, memories, support, admin_support, reports, report_templates, welli
  agents/             — extractor (self-validating), escalation, graph (LangGraph), rag tools, prompts
  pipeline/           — orchestrator, page_streamer, page_router, preprocessing, postprocessor, classifier, batch_orchestrator
  retrieval/          — evidence-tier weighting (artifact-class), not a reranker
  vectorstore/        — chunker, embedder, entity_indexer, image_indexer, qdrant_store
  domain/             — entity_resolver, formations, units, well_taxonomy
  storage/            — local_storage (filesystem), redis_store
  workers/            — celery_app, tasks/ingest (+ batch submit/poll/collect), tasks/report, tasks/memory
  db/                 — engine (asyncpg), models (SQLAlchemy 2.0), queries/
  utils/              — gemini_client, logging, rate_limit, sse, pricing
  tools/              — bootstrap_super_admin, api_smoke, qdrant_reset, dump_openapi, probe_chat
```

## Where to look first when debugging
- 4xx auth/permission → [02-auth-and-rbac.md](02-auth-and-rbac.md)
- Ingest stuck/failed → [03-ingestion-pipeline.md](03-ingestion-pipeline.md) + `wellsynthai-celery-worker` logs
- Wrong/empty chat answer → [04-retrieval-and-chat.md](04-retrieval-and-chat.md)
- Missing page image / viewer lag → [13-document-viewer-and-serving.md](13-document-viewer-and-serving.md)
