# WellSynth AI Backend: Knowledge Transfer and Project Handover

> **Prepared:** 2026-10-01 · **Branch at time of writing:** `feat/ddr-eowr-linkage`
> (1 commit ahead of `origin/main`, which was last moved by the prod CI bump to image `7018`)
> · **Remote:** Azure DevOps `AIGENITY/WellsynthAI`

This is the starting point for whoever takes over this repository. It goes through
the repo package by package and module by module, and covers the end-to-end
flows, the data model, deployment, and the known risks. It does **not** replace
the detailed docs. Each section links to the file that owns the detail. Read this
document first, then the per-package `CLAUDE.md` for whatever you are about to change.

**How the docs fit together**

| Doc | Use it for |
|---|---|
| **This file** | Orientation, the module map, flows, risks, and the handover checklist |
| [`README.md`](README.md) | Quickstart, tenants and billing concepts, migration notes |
| [`wiki/`](wiki/README.md) | The engineering reference: one page per subsystem |
| `app/**/CLAUDE.md` | Per-package operating rules: invariants, traps, measured tuning history |
| `app/**/README.md` | Per-package prose overviews (shorter than the `CLAUDE.md` files) |
| [`frontend_docs/communication_from_backend.md`](frontend_docs/communication_from_backend.md) | The authoritative backend→frontend wire contract |
| [`architecture/index.html`](architecture/index.html) | Offline architecture atlas you can click through (open it in a browser) |

> **Code is the source of truth.** Where a doc and the code disagree, trust the code.
> §13 lists the doc drift found while writing this.

---

## Contents

1. [TL;DR](#1-tldr)
2. [What the product does](#2-what-the-product-does)
3. [System architecture](#3-system-architecture)
4. [Repository map](#4-repository-map)
5. [Package-by-package walkthrough (`app/`)](#5-package-by-package-walkthrough-app)
6. [End-to-end flows](#6-end-to-end-flows)
7. [Data model](#7-data-model)
8. [API surface](#8-api-surface)
9. [Configuration and feature flags](#9-configuration-and-feature-flags)
10. [Environments, deployment and CI/CD](#10-environments-deployment-and-cicd)
11. [Migrations](#11-migrations)
12. [Testing and tooling](#12-testing-and-tooling)
13. [Known risks, tech debt and doc drift](#13-known-risks-tech-debt-and-doc-drift)
14. [Handover checklist](#14-handover-checklist)
15. [Glossary](#15-glossary)

---

## 1. TL;DR

- **What it is:** a multi-tenant FastAPI and Celery backend for oil-and-gas well-report
  intelligence. It ingests PDFs (and WITSML and spreadsheets), extracts page markdown,
  tables, images and entities with Gemini, indexes them in Qdrant, and serves grounded
  chat, search, strategic reports, NPT/lessons dashboards, a time-depth drilling
  benchmark, root-cause analysis, and DDR↔EOWR event linkage.
- **Stack:** Python 3.11 · FastAPI · Celery (Redis) · Postgres 16 (SQLAlchemy 2 async and
  Alembic) · Qdrant · local filesystem storage · Google Gemini (`google-genai`) ·
  LangGraph · PyMuPDF and LiteParse · structlog and OpenTelemetry → Azure Monitor.
- **Runs as:** one Docker Compose stack per environment (`local`, `dev`, `prod`) on an
  Azure VM managed through Portainer. Images are built by Azure DevOps pipelines and
  pushed to ACR `dsaishared.azurecr.io/wellsynthai-backend`.
- **Hard rules that keep the system correct** (from [`app/CLAUDE.md`](app/CLAUDE.md)):
  1. All runtime knobs live in `app/config/`. Never call `os.getenv` elsewhere.
  2. Tenant scoping is a security boundary. Every Qdrant query filters on
     `project_id IN accessible_project_ids(...)`, and every SQL read is caller-scoped.
  3. Routes validate and dispatch. Heavy work (LLM calls, rasterization, reports) runs in Celery.
  4. Agents never decide access. They receive already-authorized scopes.
  5. Credit settle/release and telemetry paths never raise into the caller.
- **Fix first** (details in §13): literal production secrets are committed in
  `docker-compose.yml` and `docker-compose.dev.yml`. No `celery beat` process
  is deployed, so the scheduled cleanup and hold sweeper never run. `deploy/` and
  `docs/` are gitignored but referenced by the docs.

---

## 2. What the product does

WellSynth AI turns well-engineering documents into searchable, cross-checked knowledge.
The documents include End of Well Reports (EOWR), Daily Drilling Reports (DDR),
drilling programs, completion reports and HSE documents.

| Capability | User-facing surface | Backend owner |
|---|---|---|
| Document ingestion (two lanes: `realtime`, `batch`) | Upload, job progress | `routes/ingest.py` → `workers/tasks/ingest.py` → `pipeline/` |
| Document viewer (page PNGs, crops, markdown) | Viewer | `routes/documents.py`, `routes/_documents_serving.py` |
| Grounded chat (SSE streaming, citations, page images, memory) | Chat | `routes/chat.py` → `agents/rag/` |
| Semantic search | Search | `routes/search.py` → `agents/rag_tools.py` |
| Strategic reports (templated, agentic, PDF/DOCX export) | Reports | `routes/reports.py` → `workers/tasks/report.py` → `agents/reports/` |
| Well Knowledge Layer (NPT events, lessons, top-5 rollups) | Insights dashboard | `pipeline/insight_*` → `routes/insights.py` |
| Document Insight Layer (verified findings, corrections) | Knowledge cards | `workers/tasks/knowledge.py` → `agents/knowledge/` |
| Time-depth benchmark / ILT (best composite, delta, CPR, aftershock, attention list) | Benchmark dashboard | `pipeline/time_depth.py`, `composite.py`, `delta.py`, … → `GET /projects/{id}/benchmark` |
| Root-cause analysis (RCA) with allow-listed web research | RCA panel | `pipeline/rca*.py` → `POST /projects/{id}/rca` |
| DDR↔EOWR linkage (merge DDR workbook, explain rows from the EOWR) | Linkage runs and export | `routes/linkage.py` → `workers/tasks/linkage.py` → `app/linkage/` |
| Multi-tenancy (orgs, roles, project sharing) | Admin console | `auth/`, `routes/admin_*.py` |
| Credits and billing (reserve → settle, wallets, alerts, upgrade queue) | Usage meter, admin billing | `billing/`, `db/queries/credits/` |
| Notifications inbox | Bell | `routes/notifications.py`, `db/queries/notifications.py` |
| Support tickets, feedback-for-tokens, book-a-demo | Support, feedback | `routes/support.py`, `feedback.py`, `demo.py` |
| Marketing blog (public read, super_admin authoring) | Public site | `routes/blog.py`, `routes/admin_blog.py` |
| Welli: public landing-page assistant (no data access) | Marketing site | `routes/welli.py`, `app/welli/SKILL.md` |

**Users and roles.** `super_admin` (platform operator, billing-exempt) · `admin` (manages
one organization) · regular seats: `drilling_engineer`, `geologist`, `asset_manager`, and
`qa_steward`. Public demo self-signup is supported. B2B onboarding is invite-only.
See [`wiki/19`](wiki/19-user-and-org-management-explained.md) and [`wiki/02`](wiki/02-auth-and-rbac.md).

---

## 3. System architecture

```text
                     Internet
                        │  HTTPS (Let's Encrypt, api.wellsynth.ai / api-dev.wellsynth.ai)
                 ┌──────▼──────┐
                 │   nginx     │  prod nginx also forwards api-dev → nginx-dev over the
                 └──────┬──────┘  shared external docker network "edge"
                        │
          ┌─────────────▼─────────────┐
          │  app  (uvicorn, 2 workers)│  FastAPI: middleware → routers
          │  app.state: GeminiClient, │  GZip ▸ CORS ▸ Auth ▸ RequestLogging
          │  GeminiEmbedder, Qdrant   │
          └──┬────────┬────────┬──────┘
             │        │        │ enqueue (Redis broker)
   ┌─────────▼─┐ ┌────▼────┐ ┌─▼──────────────────────────────────────────────┐
   │ Postgres16│ │  Redis  │ │ Celery fleets (separate containers)            │
   │ durable   │ │ broker, │ │  celery-worker          -Q extract,rasterize,  │
   │ truth     │ │ chat STM│ │                            reports,default (c=4)│
   └─────▲─────┘ │ limits, │ │  celery-batch-worker    -Q batch_poll,         │
         │       │ locks   │ │                            batch_collect (c=2) │
         │       └─────────┘ │  celery-knowledge-worker -Q aggregate (c=2)    │
         │                   └──┬─────────────────────────────────────────────┘
         │                      │
   ┌─────┴──────┐   ┌───────────▼───┐   ┌──────────────────────────────┐
   │ Qdrant     │   │ Local FS      │   │ Google Gemini                │
   │ per-org    │   │ storage_root: │   │ flash-lite / flash / 3.7-flash│
   │ collections│   │ pdfs/pages/   │   │ embeddings, Batch API,       │
   │            │   │ artifacts     │   │ Google Search grounding      │
   └────────────┘   └───────────────┘   └──────────────────────────────┘
   + Resend (email) · Azure Application Insights (telemetry) · Google OAuth
```

**State ownership.** Postgres is the only store that cannot be rebuilt. Redis (broker,
chat short-term memory, rate limits, debounce locks) and Qdrant (vectors) can both be
rebuilt from Postgres plus storage. Local FS holds the original PDFs, page PNGs and
per-page artifacts.

**Concurrency model.** The one `GeminiClient` per process enforces
`Semaphore(max_concurrent_pages=60)` along with adaptive 429 pacing. Building a client
per request removes the cap. The cluster ceiling is roughly
`max_concurrent_pages × processes`. Prod runs 2 uvicorn workers plus the worker
concurrency shown above.
Celery runs `prefetch_multiplier=1`, `acks_late`, `max_tasks_per_child=5`, and a
per-child memory cap. Each task runs its async body in a fresh event loop and disposes
the SQLAlchemy engine on exit (asyncpg connections are bound to one event loop).

**Startup side effects** (`app/main.py` lifespan, each one only warns on failure):
Qdrant `ensure_collection()` → mark stuck `queued`/`running` reports `failed` and release their
credit holds → idempotent super_admin seed → sync the free-plan token budget into the DB.

---

## 4. Repository map

| Path | What it is | Tracked? |
|---|---|---|
| `app/` | The backend package (~75k lines). See §5 | ✅ |
| `tests/` | ~85 pytest modules, plus `fixtures/sajaa37_attachment_truth.json` | ✅ |
| `alembic/`, `alembic.ini` | Alembic environment. **Revisions 001–029, head `029`** | ✅ |
| `migrations/` | Numbered raw SQL `012…040` plus `init.sql` (fresh-volume baseline) | ✅ |
| `wiki/` | Engineering wiki, pages 01–22 plus `LOW_LEVEL_DESIGN.md` | ✅ |
| `frontend_docs/` | FE contracts, feature plans, `openapi.json` (regenerate after route changes) | ✅ |
| `architecture/` | Static HTML architecture atlas, with a citation index in its README | ✅ |
| `Dockerfile` | `python:3.11-slim` with PyMuPDF and WeasyPrint system deps, `uvicorn app.main:app :8080` | ✅ |
| `docker-compose.local.yml` | Local dev stack (builds the image, adds a `langgraph-dev` service) | ✅ |
| `docker-compose.dev.yml` / `docker-compose.yml` | Dev / prod stacks (pinned ACR image tags that CI rewrites) | ✅ |
| `azure-pipelines.dev.yml` / `.prod.yml` | Build → push to ACR → rewrite the compose tag → commit back (`dev` / `main`) | ✅ |
| `nginx.conf` / `nginx.dev.conf` | Reverse proxy and TLS config for the prod and dev VMs | ✅ |
| `langgraph.json` | `langgraph dev` entry: `report_section_agent` → `app/agents/reports/dev_graph.py` | ✅ |
| `requirements.txt` / `requirements-lock.txt` | Runtime deps (`liteparse==2.11.0` **must stay pinned**) | ✅ |
| `pyproject.toml` | ruff (line 100; `E,F,I,N,W,UP`) and pytest (`asyncio_mode=auto`) | ✅ |
| `scripts/live_discovery.py`, `probe_extraction.py` | One-off research harnesses (template discovery, standalone extraction probe) | ✅ |
| Root `*_PLAN.md`, `ARCHITECTURE_REVIEW.md`, `CODEBASE_AUDIT_BLACKBOARD.md`, `COST_ANALYSIS.md`, `PROVIDER_OPTIONS.md` | Design history and audits from 2026-07/08. Useful for the *why*, but may be partly superseded | ✅ |
| `blog/` | Draft marketing blog post(s) | ✅ |
| `.github/` | Copilot instructions and an audit agent prompt (the repo itself lives on Azure DevOps) | ✅ |
| `deploy/` | `dev-autopull.sh` (+ systemd unit/timer), `renew-ssl.sh`, `migrate_qdrant_collections.py` | ❌ **gitignored** |
| `docs/` | Benchmarking FR specs, blog API contract (referenced from the wiki and code) | ❌ **gitignored** |
| `data/` | Local runtime data and `data/plan_documentations/` (historical plans) | ❌ gitignored |

> ⚠ `deploy/` and `docs/` exist on the original developer's machine but **not in the
> repository**. Before handover, either commit them or copy them to a shared location.
> See §14.

---

## 5. Package-by-package walkthrough (`app/`)

Every subpackage has a `CLAUDE.md` (rules and traps) and a `README.md` (overview). The
table below is the map. The subsections give what you need before opening the code.

| Package | Lines (approx.) | One-line role |
|---|---|---|
| [`main.py`](app/main.py) | 380 | App factory, middleware order, router registration, lifespan |
| [`config/`](app/config/__init__.py) | 1.2k | `Settings`, composed from 7 mixins. The only home for knobs |
| [`auth/`](app/auth/CLAUDE.md) | 2.3k | JWT/bcrypt, middleware, project and org ACL, email, seed |
| [`routes/`](app/routes/CLAUDE.md) | 12k | HTTP surface (30 modules) |
| [`billing/`](app/billing/CLAUDE.md) | 160 | `require_credits` dependency, estimates, demo IP gate |
| [`db/`](app/db/CLAUDE.md) | 9k | ORM models (46 tables), async engine, `queries/` |
| [`models/`](app/models/CLAUDE.md) | 160 | Pipeline Pydantic shapes and the LangGraph `PageState` |
| [`agents/`](app/agents/CLAUDE.md) | 14k | Extraction graph, RAG chat, reports, knowledge agent |
| [`pipeline/`](app/pipeline/CLAUDE.md) | 13k | Ingestion phases 0–8, insights, benchmark, RCA |
| [`domain/`](app/domain/CLAUDE.md) | 4k | Oilfield vocabularies, units, DDR models and parsers (pure) |
| [`linkage/`](app/linkage/CLAUDE.md) | 5k | DDR workbook merge and EOWR event reconciliation |
| [`vectorstore/`](app/vectorstore/CLAUDE.md) | 1.4k | Chunk, embed, Qdrant client, per-org collections |
| [`retrieval/`](app/retrieval/CLAUDE.md) | 100 | Static evidence-tier weighting by `artifact_class` |
| [`storage/`](app/storage/CLAUDE.md) | 390 | `LocalStorage`, a bucket/key filesystem store |
| [`workers/`](app/workers/CLAUDE.md) | 2.6k | Celery app, queues, tasks |
| [`observability/`](app/observability/CLAUDE.md) | 800 | structlog, OTel spans/metrics, health, Celery propagation |
| [`utils/`](app/utils/CLAUDE.md) | 1.2k | Gemini client and batch, pricing, rate limit, SSE, locks |
| [`tools/`](app/tools/CLAUDE.md) | 1.5k | One-shot ops CLIs (`python -m app.tools.<name>`) |
| [`welli/`](app/welli/CLAUDE.md) | — | Prompt content only (`SKILL.md`) for the public bot |

### 5.1 `app/main.py`

- Middleware is registered `GZip → CORS → Auth → RequestLogging`. Starlette wraps in
  reverse order, so GZip is innermost. Reordering changes which error responses carry
  CORS headers.
- **Router order matters:** `insights_router` must be registered before `projects_router`.
  Otherwise the static path `/projects/insights` is captured by `/projects/{project_id}`
  and returns a 500 on the UUID cast.
- System endpoints: `GET /` (capabilities), `/health` (liveness), `/ready` and
  `/health/deep` (dependency readiness, 503 when degraded).
- A global exception handler returns JSON 500s. A custom OpenAPI generator feeds `app.tools.dump_openapi`.

### 5.2 `app/config/`

`Settings` (pydantic-settings, `.env`-backed, `extra="ignore"`) is composed from:

| Mixin | Covers |
|---|---|
| `_pipeline.py` | Gemini key, render/route thresholds, concurrency, pacing, batch lane, text repair |
| `_benchmarking.py` | FR-x benchmark feature flags, template discovery, RCA vision |
| `_billing.py` | `tokens_per_credit`, free budget, enforcement switches, estimates, alerts, demo gates |
| `_models_knowledge.py` | Extraction/escalation/insight models, knowledge-layer budgets and models |
| `_chat_report.py` | `chat_*` budgets and thresholds, report models and loop ceilings |
| `_linkage.py` | Linkage flags and models |
| `_platform.py` | DB/Redis/Qdrant URLs, JWT, OAuth, Resend, CORS, telemetry, storage, Welli |

The inline comments in these files record the tuning history (why a value is what it
is) and count as documentation. Add every new field with a comment explaining its default.
`get_settings()` builds a fresh `Settings()` on each call. Read it once per request or task.

### 5.3 `app/auth/`: identity and access control

| Module | Role |
|---|---|
| `jwt_verify.py` | Issue and verify HS256 access and refresh tokens (`TokenClaims`, `AuthError`) |
| `password.py` | bcrypt hash and verify (`bcrypt_rounds`=12) |
| `middleware.py` | `AuthMiddleware`, `AuthUser`. **`ANON_PATHS` / `ANON_PREFIXES` is the only way to make a route public.** Loads profile and effective tier on every request |
| `deps.py` | `current_user`, `require_role(...)` |
| `project_access.py` | **The single source of truth for project ACL.** `accessible_project_ids()`, `assert_project_access`, `describe_access`, `PAID_TIERS` |
| `org_access.py` | **The single source of truth for org ACL.** `assert_org_access`, `load_and_assert_org`, `resolve_org_scope`, `ORG_ASSIGNABLE_ROLES` |
| `routes.py` | `/auth/*`: login, refresh, change-password, demo signup and verify, Google OAuth callback, profile |
| `email/` | Resend templates: `account.py`, `billing.py`, `support.py`, `_render.py`. Sending domain is **wellsynth.ai** |
| `seed.py` | Idempotent super_admin bootstrap, attached to the `wellsynth_admin` org |

The project access model is the union of ownership, org-wide `shared` visibility (paid
same-org members only), and explicit `project_members` grants (`viewer`/`editor`).
Projects you cannot see return **404, not 403**, so their existence stays hidden.
**Every list endpoint must derive its filter from `accessible_project_ids` /
`resolve_org_scope`.** Hand-rolled filters have caused real cross-tenant bugs, and
`tests/test_project_access.py` and `tests/test_org_access.py` pin these rules.
Retrieval call sites must pass `role=None` into the collection resolver (see §5.10).

### 5.4 `app/routes/`: HTTP surface

Every route follows the same shape: **validate → authorize → reserve credits → persist intent →
dispatch → return**. Request and response Pydantic models live next to their route.
Metered routes take `Depends(require_credits("<feature>"))` and must settle or release
exactly once on every exit path. `chat.py`'s disconnect branch is the reference
implementation. SSE is framed with `utils/sse.sse_pack`. The full endpoint list is in §8.

Notable modules: `insights.py` (1.4k lines: dashboards, DDR, knowledge reads, RCA,
benchmark), `ingest.py` (lane selection, dedup, dispatch), `admin_organizations.py` (tenant
lifecycle and the `/overview` aggregate), `upgrade_requests.py` (two routers in one
module), `_documents_serving.py` (page PNG, crop, markdown and manifest serving).
`organizations.tier` has exactly one writer: `db.queries.credits.set_org_tier`.

### 5.5 `app/billing/` and the credit system

`billing/` is thin. `deps.require_credits(feature)` reserves at route entry and stores a
`CreditHold`. `estimates.estimate_credits` sizes the reserve. **An unknown feature reserves 0**, so a
new metered feature must be added there. `demo_guard.enforce_demo_ip` is a fail-open,
Redis-backed IP gate that applies only to demo users.

The accounting lives in `db/queries/credits/` (`_core`, `_reserve`, `_settle`, `_ledger`,
`_admin`, `_alerts`, `_summaries`). Invariants, in full in
[`app/db/queries/CLAUDE.md`](app/db/queries/CLAUDE.md):

- 1 credit = `tokens_per_credit` tokens (100,000). Only `usage_events` stores raw tokens.
- Reserve writes a `credit_holds` row. settle/release *claim* it once (`FOR UPDATE`).
  `ref_id` (job/report/`knowledge:{doc}`) is the idempotency key for worker-backed work.
- Lock order: `credit_holds → user_subscriptions → organizations`.
- Every balance mutation writes a `credit_ledger` row **in the same transaction**,
  including admin top-ups, activation and tier changes.
- `free_tier_enforced` (True) gates the free tier. `billing_enforcement_enabled`
  (**False**) gates paid tiers. Paid-org members are never balance-blocked. They trigger
  1×/2×/3× threshold alerts instead (soft cap).
- `super_admin` is billing-exempt.
- Worker-side `record_usage` calls must pass `organization_id=` explicitly. Otherwise
  the spend lands in the `unattributed` bucket.

Plain-English walkthroughs: [`wiki/14`](wiki/14-billing-demo-and-feedback.md), [`wiki/18`](wiki/18-credits-and-billing-explained.md).

### 5.6 `app/db/`: persistence

- `engine.py`: asyncpg engine (prefers `worker_database_url`, forces the `+asyncpg` scheme),
  `pool_size=10, max_overflow=20` per process, `expire_on_commit=False`.
- `models/`: ORM split by domain (`identity`, `projects`, `documents`, `billing`,
  `memory_chat`, `notifications`, `support`, `reports`, `blog`, `well_facts`, `ddr`,
  `knowledge_cards`, `linkage`). See §7 for the table list.
- `queries/`: one module per feature. Each opens its own short-lived session, returns plain
  dicts or dataclasses, and scopes every read by user, project or org. Never hold a row lock across an
  LLM or HTTP call.
- `migration_sql.py`: `exec_sql()` splits multi-statement SQL for asyncpg (used by Alembic).
- `document_insights.raw_npt_events` / `raw_lessons` are a **frozen baseline** that the
  knowledge agent writes once before its first correction. They are not a working copy.

### 5.7 `app/models/`

`schemas.py` holds the pipeline data contracts (`PageExtraction`, `PageBaseline`, `PageImage`,
`TableResult`, `ImageResult`, `Entity`, `JobStatus`). Prefer adding optional fields,
because persisted `page_NNN.state.json` artifacts are never migrated.
`state.py` holds the LangGraph `PageState` (`iteration` 1/2/3, `final` is what downstream
reads, `png_bytes` must never be logged). Not to be confused with `app/db/models/`.

### 5.8 `app/agents/`: LLM agents

Two unrelated concerns share this package:

**Per-page extraction**, called by the pipeline: `graph.py` (LangGraph: `extractor` →
END | `escalation` → END), `extractor.py`, `escalation.py`, `extraction_parse.py` (**all**
model JSON goes through this: never `json.loads` inline), `prompts.py`. Safety filters are
disabled for extraction because well reports trip "dangerous content" filters.

**Synthesis agents**, which share the retrieval tools in `rag_tools.py` (`search_chunks`,
`search_entities`, `search_images`; `_scope_filters` **raises on an empty project set**;
governance and tier weighting are applied per query):

- **`rag/`: chat.** The pipeline is deliberately two layers: `prefilter` (regex, free) →
  `guard` (one flash-lite call that answers workspace/meta questions or delegates) →
  `synthesize` (flash with function calling: `rag_search`, `fetch_document_outline`,
  `fetch_document_knowledge`, `fetch_project_knowledge`, `fetch_project_insights`,
  `fetch_table`). The closing answer turn **replays the tool conversation**. Cited page
  PNGs are attached on every grounded turn. Memory has three parts: Redis STM, the Postgres
  transcript, and opt-in long-term memories. `tests/test_chat_pipeline_contracts.py` pins
  the SSE frame contract (`done → verification_update → followups_update`).
  Required reading: [`app/agents/rag/CLAUDE.md`](app/agents/rag/CLAUDE.md).
- **`reports/`: strategic reports.** `report_agentic_enabled=True` uses the agentic path
  (`master.py` plans, delegates, reviews and composes over per-section LangGraph analysts in
  `section_agent.py`/`agent_nodes.py`, sharing a `blackboard.py`, with `retrieval_tool.py`
  tool loops and optional Google Search grounding tagged `[external]`). `False` uses
  the legacy single-shot `builder.py`, which is the **rollback path, so keep it working**.
  Templates come from `templates.py` (built-in) and `registry.py` (DB custom). `template_generator.py`
  turns a plain-English brief into a spec. PDF (WeasyPrint) and DOCX (python-docx) are rendered
  on demand from stored markdown. `activity.py` feeds the live agent-activity view.
- **`knowledge/`: Document Insight Layer.** A post-ingest, open-book second opinion on
  Phase 6: question bank (`question_bank.py`, deterministic `question_gen.py`) →
  bounded retrieval loop (reused from `reports/retrieval_tool` through an additive seam) →
  **computed** confidence (`card.py`) → five correction gates including a deterministic
  verbatim-quote check (`corrections.py`) → `document_knowledge_cards` and optional
  Layer A write-back. Hard-capped by `budget.py` (`knowledge_max_total_llm_calls`=45, with a
  compact tier for short docs). `cross_doc.py` does project-level synthesis and **never writes
  back**. `ddr_extract.py` is now a *fallback* DDR path that is skipped when deterministic
  rows exist.

Model tiering is consistent across agents: **flash-lite** for routing, grading,
titles and STM; **flash** (`gemini-3.5-flash`) for synthesis and reasoning;
**`gemini-3.7-flash`** for linkage. Model ids are always `Settings` fields.

### 5.9 `app/pipeline/`: ingestion and analytics

Entered only from Celery (`workers/tasks/ingest.py`). `orchestrator.PipelineOrchestrator`
(with `_orch_run`, `_orch_phases`, `_orch_finalize`, `_worker`, `_job_result`) runs:

| Phase | `jobs.current_phase` | Module(s) |
|---|---|---|
| pre | `parsing_domain_source` | `source_adapters.py` (WITSML/XML and others rendered as pages) |
| 0 / 0.5 | `preprocessing` | `preprocessing.py` (PyMuPDF fast pass and LiteParse native markdown), `text_repair.py` (constant-offset font-corruption repair) |
| 1 | — | `page_streamer.py` (streaming rasterizer, `render_concurrency=8`, waits on `baseline_ready`) |
| 2 | `streaming_extraction` | `page_router.py` (VISION / LITEPARSE / CLASSIFY), `classifier.py`, `agents/graph.py`, `extraction_guard.py`, `numeric_guardrail.py`, `entity_normalizer.py` |
| 3 | `postprocessing` | `postprocessor.py` (cross-page table merge, the single markdown normalization seam `_build_page_map`), `markdown_normalizer.py` (idempotent) |
| 4 / 4b / 4c | `vectorizing` | `vectorstore/` chunk, embed and upsert; entities; images |
| 5 | — | `artifact_classifier.py` (only when the user left `artifact_class='other'`) |
| 6 | `insight_synthesis` | `insight_extraction.py` (ontology-driven, about 5 pages per part, in parallel) |
| 7 | `insight_projection` | `insight_projector.py` (Layer A → Layer B `npt_events`/`lessons`) |
| 7b | `ddr_projection` | `ddr_pdf_lane.py` (deterministic PDF DDR), `ddr_projector.py` (`DailyReport` → `daily_reports`/`activity_intervals`) |
| 8 | `insight_rollup` | `insight_rollup.py`, `_rollup_compute.py` (project top-5 NPT and lessons) |

**Phases 0–4 are load-bearing. Phases 5–8 are best-effort and must swallow failures.**
The batch lane (`batch_orchestrator.py`, using `utils/gemini_batch.py`) has no escalation loop and
must call the full `extract()`, not `extract_fast`.

**Benchmark and ILT analytics** (deterministic, no LLM unless noted): `time_depth.py`
(cumulative curve and time attribution), `composite.py` (FR-3 best composite keyed by
section × depth band × activity), `delta.py` (FR-4 scope/NPT/ILT decomposition),
`performance_ratio.py` (FR-8 CPR), `aftershock.py` (FR-11), `response_effectiveness.py`
(FR-10, evidence only and never advice), `attention.py` (FR-12), `benchmark_payload.py` (the one
payload builder shared by the route and the cache), `benchmark_cache.py` (FR-5 recompute,
`source_hash`), `template_discovery.py` (vision describes an unknown DDR *layout*, and code
computes every number).

**RCA:** `rca.py`, `rca_evidence.py` (benchmark-derived evidence), `rca_research.py` (web
research hard-filtered to `RCA_ALLOWED_DOMAINS`), `rca_cache.py` (FR-6 persistence in
`rca_runs`).

Required reading before you touch routing or extraction: [`app/pipeline/CLAUDE.md`](app/pipeline/CLAUDE.md)
(it records measured failure modes: single-figure pages, vector schematics, broken
ToUnicode CMaps, offset-corrupted text, and blank-page nets).

### 5.10 `app/vectorstore/` and `app/retrieval/`

- Three logical kinds: **chunks**, **entities** and **images**. Physical names are per org:
  `wellsynthai_org_<uuid_hex>_{documents,entities,images}`, and the default `wellsynthai_*` set
  is used for no-org callers and legacy data. Write with `org_id_slug` only. `org_slug` (name-based) is
  a read fallback.
- Every point payload carries `user_id, role, organization(_id), project_id, document_id`.
  Payload role and org are snapshots and must **never** be used for authorization.
- `chunker.py` (1000 characters with 200 overlap), `embedder.py` (`gemini-embedding-001`, 768 dims,
  batch 20, `embedding_max_concurrent_batches=4`: the embedding endpoint hits 429 first).
- **Trap:** never call `for_org_id(org_id, user.role)` at a retrieval call site. Passing
  `super_admin` routes to the empty default collections. Pass `role=None`.
- Changing the embedding model or dimensions means a full re-index (`app.tools.qdrant_reset`
  followed by re-ingestion).
- `retrieval/tier_weighting.py`: `score = raw_cosine × weight(artifact_class)` (default
  0.5). It is static and is not a reranker. `artifact_class` is looked up from Postgres per query
  rather than stored in the payload, so class edits apply immediately.
  `chat_score_threshold` is compared in **raw** space and `chat_evidence_strong_score` in **weighted** space.

### 5.11 `app/domain/`: oilfield knowledge (pure)

No I/O, no LLM and no DB, with one exception: `well_registry.py`, which is **disabled**. String similarity
mis-merges pad wells such as Sajaa 6 and Sajaa 9.

| Module | Role |
|---|---|
| `ontology.py` / `_ontology_registry.py` | The extraction contract: node and edge types, `validate_ontology()` |
| `npt.py` | **Closed** NPT category vocabulary, `canonicalize()` |
| `drilling_problems.py` | Closed problem-signal vocabulary (FR-10) |
| `well_taxonomy.py` | **Open** entity vocabularies (hints, not whitelists) |
| `units.py` | Unit conversion. `is_sane()` drives the numeric guardrail |
| `formations.py`, `entity_resolver.py` | Formation aliases, in-memory entity dedupe |
| `ddr.py` / `_ddr_model.py` | `DailyReport`/`Activity`, `parse_witsml_ddr`, `classify_activity`, `normalize_well_name` |
| `ddr_canonical.py` | The canonical DDR information model that everything downstream reads |
| `ddr_profile.py`, `ddr_pdf.py` | Declarative PDF DDR template profiles and the deterministic PDF parser (FR-1) |
| `ddr_intraday.py` | Detects "afternoon update" snapshots (`source_tier="intra_day"`, kept out of the benchmark) |
| `tabular_profile.py` | Declarative spreadsheet DDR layouts (used by linkage) |

Rule: **layout is data.** A new operator's DDR form gets a new profile constant, not parser edits.

### 5.12 `app/linkage/`: DDR↔EOWR event linkage

Merges a tabular DDR workbook (raw day tabs or an already-merged export) into one lossless
row grid, then reconciles it against the EOWR in **both directions**:

- `workbook.py`, `merge.py`, `context.py` (BHA state machine: explicit / carried / inferred),
  `models.py` (`DdrRow`, stable `row_uid` from the cell address).
- `events/`: `npt_tables.py` (EOWR NPT ledger plus the accounting invariant),
  `ddr_detector.py` (rule-based **reverse** path), `narrative.py` (the only LLM stage,
  `gemini-3.7-flash`, about $0.02 per EOWR), `cache.py` (content-addressed replay for
  determinism), `subtypes.py` (closed vocabulary).
- Matcher: `indexes.py` (blocking on observed fields only), `fusion.py` (Fellegi–Sunter),
  `align.py` (anchor-and-expand), `rerank.py` (constraints → LLM judge).
- `reconcile.py` produces `CONFIRMED | PROBABLE | UNEXPLAINED | ORPHANED_NPT`. `export.py` writes the
  annotated workbook (idempotently). `evaluation.py` holds the held-out-by-leg metrics.
- Corpus wells: Sajaa 37 and Sajaa 1. **Quote held-out numbers, not in-sample ones.** The
  measured history and the dead ends are recorded in [`app/linkage/CLAUDE.md`](app/linkage/CLAUDE.md).
- Persisted in `linkage_runs`, `ddr_rows`, `linkage_annotations`, `linkage_corrections`.
  Users can correct a row with `POST …/rows/{row_uid}/correct`.

### 5.13 `app/storage/`

`LocalStorage` (async `aiofiles`) uses the path contract
`<storage_root>/<bucket>/<user_id>/<project_id>/<document_id>/<filename>`, with buckets
`pdfs` (`source.pdf`), `pages` (`page_NNN.png`) and `artifacts` (markdown, tables, `page_NNN.state.json`).
Always build keys with `_key(...)`. A delete removes the whole document subtree **and** fans
out a Qdrant delete. The API mirrors the old `SupabaseStorage`, so it can be swapped for an object store later.

### 5.14 `app/workers/`: Celery

| Queue | Tasks | Served by |
|---|---|---|
| `extract` | `ingest_document` (realtime lane) | `celery-worker` (c=4) |
| `rasterize` | `ingest_batch_submit` | `celery-worker` |
| `reports` | `report.generate` | `celery-worker` |
| `default` | `extract_user_memory`, `cleanup.purge_stale_demo`, `cleanup.sweep_credit_holds` | `celery-worker` |
| `batch_poll` | `poll_batch_job` (self-reschedules every 300s, gives up at 30h) | `celery-batch-worker` (c=2) |
| `batch_collect` | `collect_batch_job` | `celery-batch-worker` |
| `aggregate` | `knowledge.interrogate_document`, `knowledge.cross_document`, `benchmark.recompute_project`, `linkage.run` | `celery-knowledge-worker` (c=2) |

Add a new task to `include=[...]` in `celery_app.py` and give it an explicit `queue=`.
Tasks must be retry-safe and finalize credits via `settle_by_ref`/`release_by_ref`. Never
import `app.main`/`app.state` in a worker. Build clients locally instead. Worker telemetry uses
`service_name=wellsynthai-worker` (configured per prefork child on `worker_process_init`).
Project-level recomputes are debounced with `utils/redis_lock.py` (lock + dirty flag + one
self-redispatch, fail-open).

### 5.15 `app/observability/`

`logging.py` (structlog), `context.py` (contextvars shared by logs, spans and usage rows),
`spans.py`/`metrics.py` (no-op without OTel), `telemetry.py` (Azure Monitor, guarded once per
process), `middleware.py` (request logging), `celery.py` (trace propagation, queue depth),
`redaction.py`, `health.py`. Rules: telemetry never fails a request; **log event names, not
sentences**; **never log prompts, chunks, document text, answers, or secrets.**

### 5.16 `app/utils/`

`gemini_client.py` (**one per process**: semaphore, 120s timeout, adaptive 429 pacing,
safety settings), `gemini_batch.py`, `pricing.py` (**the only place $/token rates live**. An
unknown model prices at $0. Note the `gemini-3.7-flash` price increase on 2026-12-31),
`rate_limit.py` (Redis token bucket, **fail-open by design**), `redis_lock.py`, `sse.py`,
`net.client_ip()` (the only approved way to read the client IP behind nginx), `email_validate.py`.
This is a leaf layer and must never import routes, agents or pipeline.

### 5.17 `app/tools/`: operations CLIs

`python -m app.tools.<name>`. Nothing in the app imports these.

| Script | Purpose | Risk |
|---|---|---|
| `apply_migrations` | Apply numbered `migrations/*.sql` idempotently | writes schema |
| `check_schema_parity` | Verify the live DB covers the ORM | read-only |
| `bootstrap_super_admin` | Create a super_admin | writes a user |
| `dump_openapi` | Regenerate `frontend_docs/openapi.json` | safe |
| `api_smoke` | Smoke test against a live API | hits a real environment |
| `probe_chat` | Run the chat agent in-process against a real corpus | spends tokens |
| `reproject_ddr`, `retier_ddr_rows` | DDR fact backfills (dry-run by default) | updates rows |
| `check_file_length` | Enforce the 400-line module budget | safe |
| `purge_demo` | Delete stale unverified demo accounts | **destructive** |
| `qdrant_reset` | Wipe and recreate Qdrant collections | **destructive** |

These scripts read the same `.env` as the app. Check `DATABASE_URL`/`QDRANT_URL` before running a destructive one.

### 5.18 `app/welli/`

Content only. `SKILL.md` is the system prompt for the public landing-page bot. It has **no**
access to user data. Edits go live for every visitor on restart, so review them like marketing copy.
Rate-limited to 40 turns per IP per 2 days.

---

## 6. End-to-end flows

### 6.1 Ingestion (realtime lane)

```text
POST /ingest/upload (multipart, project_id, lane, artifact_class)
 → auth + assert_project_access(ingest) → require_credits("ingest") (per-MB, halved for batch)
 → content-hash dedup → documents + jobs rows → store source.pdf
 → Celery ingest_document [extract]
     Phase 0–4 (load-bearing) → Phases 5–8 (best-effort)
     → jobs.current_phase updated every 3s (the FE polls GET /ingest/{job_id})
     → usage_events row (with organization_id) → settle_by_ref(job_id)
     → dispatch knowledge.interrogate_document [aggregate] and benchmark.recompute_project [aggregate]
```

Batch lane: `ingest_batch_submit` → `poll_batch_job` (loop) → `collect_batch_job`. The
bookkeeping lives in `jobs.batch_meta`. It is roughly 50% cheaper and can take up to about 24h.

### 6.2 Chat

```text
POST /chat/stream → require_credits("chat") → accessible_project_ids + context card
 → run_agent_stream: prefilter → guard (flash-lite) → synthesize (flash + tools, ≤ chat_max_rag_calls rag_search)
 → SSE frames: thought / token / … / done → verification_update → followups_update
 → settle on success; release on error or CancelledError (client disconnect)
 → transcript persisted; STM updated in the background; extract_user_memory dispatched
```

### 6.3 Reports

`POST /projects/{id}/reports` → resolve the template (section count sizes the reserve) →
`strategic_reports` row `queued` → `report.generate` [reports] → master plans → section
agents (plan → gather → reason → draft → reflect, ≤2 reflections, ≤6 retrieval calls) →
compose → markdown stored → `settle_by_ref(report_id)`. Exports are rendered on demand at
`/pdf` and `/docx`. Crash recovery runs at API startup.

### 6.4 Knowledge layer, benchmark, RCA, linkage

- **Knowledge:** after ingest, the worker reserves `knowledge:{document_id}`, runs the bounded agent,
  writes the card and any corrections (re-running Phases 7 and 8 for that document), and settles. Reads
  (`GET /documents/{id}/knowledge`, …) are unmetered and never recompute.
- **Benchmark:** recomputed by ingestion, never by a dashboard read. `GET
  /projects/{id}/benchmark` serves the cached default payload, or reduces live when query
  params are set. Both paths share `benchmark_payload.build_benchmark_payload`.
- **RCA:** `POST /projects/{id}/rca` assembles evidence (NPT, lessons, benchmark cells), runs
  allow-listed web research and reasoning, and caches the result in `rca_runs`.
- **Linkage:** `POST /projects/{id}/linkage/runs` (DDR workbook plus EOWR document) →
  `linkage.run` [aggregate] → rows, annotations, export.

### 6.5 Tenant and billing lifecycle

super_admin creates an org (`POST /admin/organizations`, the slug is frozen) → activates it
(seeds the wallet, writing a ledger row) → the org admin invites members (bounded by `user_quota`) →
members consume credits from the org pool (paid tiers) or their user pool (free) → threshold
alerts go out by email and notification → free users can file `POST /me/upgrade-request` →
super_admin accepts it (tier, activation and seed credits in one transaction).

---

## 7. Data model

46 tables, defined in `app/db/models/`:

| Area | Tables |
|---|---|
| Identity and tenancy | `organizations`, `users`, `users_profile` |
| Projects | `projects`, `project_members` |
| Documents and ingestion | `documents`, `jobs`, `page_extractions`, `usage_events` |
| Billing | `subscription_plans`, `user_subscriptions`, `credit_ledger`, `credit_holds`, `user_feedback` |
| Chat and memory | `chat_sessions`, `chat_messages`, `user_memories` |
| Notifications and upgrades | `user_notifications`, `upgrade_requests`, `demo_requests` |
| Support | `support_tickets`, `support_ticket_attachments`, `support_ticket_notes` |
| Reports | `report_templates`, `strategic_reports` |
| Well Knowledge Layer | `wells`, `formations`, `document_insights` (Layer A), `npt_events`, `lessons`, `fact_edges`, `well_insights`, `project_insights` |
| DDR and benchmark | `daily_reports`, `activity_intervals`, `ddr_quarantine`, `project_benchmarks`, `rca_runs` |
| Document Insight Layer | `document_knowledge_cards`, `project_knowledge_cards` |
| Linkage | `linkage_runs`, `ddr_rows`, `linkage_annotations`, `linkage_corrections` |
| Blog | `blog_posts`, `blog_media` |

Reference: [`wiki/06-database-schema.md`](wiki/06-database-schema.md),
[`wiki/17-well-knowledge-layer.md`](wiki/17-well-knowledge-layer.md).

---

## 8. API surface

All routes require a JWT unless they are listed in `ANON_PATHS`/`ANON_PREFIXES` (marked 🌐).
Swagger is at `/docs`. The machine-readable contract is [`frontend_docs/openapi.json`](frontend_docs/openapi.json).

| Area | Endpoints |
|---|---|
| Auth (`app/auth/routes.py`) | 🌐`POST /auth/login`, 🌐`/auth/refresh`, `/auth/change-password`, 🌐`/auth/demo/signup`, 🌐`/auth/verify-email`, `/auth/demo/resend-verification`, `GET /auth/me`, `PATCH /auth/me/profile`, 🌐`POST /auth/google/callback` |
| Me | `GET /me/usage`, `POST/GET /me/upgrade-request`, `POST /me/upgrade-request/cancel`, `/me/notifications` (list, `unread-count`, `{id}/read`, `read-all`, `{id}/dismiss`, `dismiss-all`) |
| Projects | `POST/GET /projects`, `GET/PATCH/DELETE /projects/{id}`, `GET /projects/{id}/shareable-users`, `GET/POST /projects/{id}/members`, `DELETE /projects/{id}/members/{uid}` |
| Documents | `GET /documents`, `GET/PATCH/DELETE /documents/{id}`, plus the page image, crop, markdown and manifest serving routes in `_documents_serving.py` |
| Ingest | `POST /ingest`, `POST /ingest/upload`, `GET /ingest`, `GET /ingest/{job_id}` |
| Search and chat | `GET /search`, `POST /chat`, `POST /chat/stream`, `GET /chat/sessions`, `PATCH/DELETE /chat/sessions/{id}`, `GET …/{id}/messages`, `POST …/{id}/archive`, `…/unarchive` |
| Memories | `GET/PATCH /memories/settings`, `GET /memories`, `DELETE /memories/{scope}/{key}`, `POST /memories/forget-all` |
| Insights, knowledge, benchmark, RCA | `GET /projects/insights`, `GET /projects/{id}/insights`, `GET /projects/{id}/ddr`, `GET /documents/{id}/ddr`, `GET /documents/{id}/insights`, `GET /documents/{id}/knowledge`, `GET /projects/{id}/knowledge`, `GET /documents/{id}/knowledge/corrections`, `POST /documents/{id}/insights/reextract`, `POST /projects/{id}/rca`, `GET /projects/{id}/benchmark` |
| Linkage | `POST/GET /projects/{id}/linkage/runs`, `GET …/runs/{run_id}`, `…/rows`, `…/annotations`, `…/export`, `POST …/rows/{row_uid}/correct` |
| Reports | `POST/GET /projects/{id}/reports`, `GET/DELETE …/{report_id}`, `GET …/{report_id}/pdf`, `…/docx`; templates: `POST /report-templates/generate`, CRUD on `/report-templates` |
| Support and feedback | `POST /support/tickets`, `GET /support/tickets/me[/{id}]`, `GET /support/tickets/{id}/attachments/{aid}`, `POST /feedback`, 🌐`POST /demo/request` |
| Public | 🌐`/blog/posts`, `/blog/tags`, `/blog/posts/{slug}`, `/blog/config`, `/blog/media/{id}`, 🌐`POST /welli/chat`, `/welli/chat/sync` |
| Admin: tenants (super_admin) | `POST/GET /admin/organizations`, `PATCH …/{id}`, `POST …/{id}/activate`, `POST …/{id}/top-up`, `GET/PUT /admin/plans[/{key}]` |
| Admin: org workspace (super_admin or that org's admin) | `GET /admin/organizations/{id}`, `…/overview`, `…/members`, `…/usage`, `…/ledger` |
| Admin: users and credits | `GET /admin/users`, `POST /admin/users/invite`, `PATCH …/{id}/role`, `…/quota`, `DELETE …/{id}`, `POST …/{id}/resend-invite`, `GET/POST /admin/users/{id}/credits`, `GET …/{id}/ledger`, `PATCH …/{id}/subscription`, `PATCH /admin/orgs/{id}/subscription` (deprecated alias) |
| Admin: projects | `POST /admin/projects/{id}/share`, `/unshare`, `/transfer`, `GET/POST …/members`, `DELETE …/members/{uid}` |
| Admin: other | `GET /admin/usage`, `GET /admin/usage/orgs`, `/admin/support/tickets[/{id}]` (GET/PATCH/DELETE), `/admin/upgrade-requests[/{id}]` plus `accept`/`reject`, `/admin/blog/*` |
| System | 🌐`GET /`, `/health`, `/ready`, `/health/deep` |

After any route change, run `python -m app.tools.dump_openapi` and update
`frontend_docs/communication_from_backend.md` if the wire format changed.

---

## 9. Configuration and feature flags

Everything lives in `app/config/` and is overridable by environment variables (`.env` locally,
the compose `environment:` block elsewhere). `.env.example` lists the core keys. Defaults
as of this handover:

| Flag | Default | Meaning |
|---|---|---|
| `free_tier_enforced` | `True` | Free users get a 402 when over budget |
| `billing_enforcement_enabled` | `False` | Paid tiers are metered but never blocked |
| `demo_immediate_access` / `demo_ip_enforced` | `True` / `True` | Demo signup logs straight in, with an IP anti-farming gate |
| `report_agentic_enabled` | `True` | Agentic reports (`False` uses the legacy single-shot path) |
| `report_web_search_enabled` / `rca_web_search_enabled` | `True` | Google Search grounding (tagged `[external]`) |
| `insight_extraction_enabled` | `True` | Phase 6 runs |
| `knowledge_layer_enabled` / `knowledge_write_back_enabled` | `True` / `True` | Post-ingest agent and Layer A corrections |
| `knowledge_free_tier_cross_doc_enabled` | `False` | Free tier skips cross-doc synthesis |
| `well_registry_enabled` | `False` | **Off on purpose** (well identity is unreliable) |
| `liteparse_ocr_enabled` | `False` | OCR is effectively unavailable (no tesseract in the image) |
| `text_repair_enabled` | `True` | Phase 0.5 font-offset repair |
| `batch_enabled` | `True` | The batch lane can be selected |
| `linkage_enabled` / `linkage_narrative_enabled` / `linkage_event_cache_enabled` | `True` | DDR↔EOWR linkage is on |
| `linkage_semantic_recovery_enabled` / `linkage_canonical_projection_enabled` | `False` | Experimental linkage tracks |
| `benchmark_*_enabled` | `True` | All FR benchmark components are on |
| `telemetry_enabled` | `True` | Azure Monitor export (needs a connection string) |

Default models: extraction `gemini-3.1-flash-lite` → escalation `gemini-3.5-flash`. Chat:
triage/lite `3.1-flash-lite`, synthesis `3.5-flash`. Reports: master/planner `3.5-flash`, grader
`3.1-flash-lite`. Linkage: `3.7-flash`. Embeddings: `gemini-embedding-001`.

---

## 10. Environments, deployment and CI/CD

| Env | Compose file | How it is updated | Hostname |
|---|---|---|---|
| local | `docker-compose.local.yml` (builds from source) | `docker compose -f docker-compose.local.yml up -d --build` | `localhost:8080` |
| dev | `docker-compose.dev.yml` | `dev` branch → `azure-pipelines.dev.yml` builds, pushes and commits the new tag → VM `deploy/dev-autopull.sh` (systemd timer) pulls and recreates the stack | `api-dev.wellsynth.ai` (via the prod nginx, `edge` network → `nginx-dev`) |
| prod | `docker-compose.yml` | `main` → `azure-pipelines.prod.yml` builds image `$(Build.BuildId)` and `prod-latest`, rewrites the tags in `docker-compose.yml`, commits `ci(prod): bump backend image to N [skip ci]` → Portainer stack update on the VM | `api.wellsynth.ai` |

Services in each stack: `postgres`, `migrate` (one-shot `alembic upgrade head`), `qdrant`,
`redis`, `app` (uvicorn `--workers 2`, `mem_limit 2g`), `celery-worker`,
`celery-batch-worker`, `celery-knowledge-worker`, `nginx` (prod) or `nginx-dev` (dev). Prod data is
bind-mounted from `/home/azureuser/wellsynth-data/…` on the VM.

**TLS:** Let's Encrypt with HTTP-01 webroot (`/var/www/certbot`), renewed by
`deploy/renew-ssl.sh` through the certbot timer. DNS is on GoDaddy. The certificate expired once
(2026-09-08) because the webroot mount was missing. The full runbook is [`wiki/22`](wiki/22-tls-and-https.md).

**Deploy checklist for a schema-changing release:** merge to `main` → wait for the prod
pipeline → update the Portainer stack → confirm the `migrate` container exited 0 → run the
smoke test (`python -m app.tools.api_smoke`) and `check_schema_parity`. Operational
procedures (bootstrap super_admin, password recovery, Qdrant reset, replaying an ingest, role
changes) are in [`wiki/10-runbooks.md`](wiki/10-runbooks.md) and [`wiki/08`](wiki/08-deployment-and-ops.md).

---

## 11. Migrations

- **Alembic is the authority** for existing databases. Revisions `001`–`029`, head **`029`**
  (`org_governance`). `001` adopts the pre-Alembic SQL history.
- `migrations/NNN_*.sql` (`012`–`040`) mirror the revisions for hand application. `init.sql` is
  the fresh-volume baseline. The numbering differs from Alembic. For example SQL `040_ddr_linkage` maps to
  Alembic `027`, and SQL `039_org_governance` maps to `029`.
- A schema change touches **three places**: `app/db/models/…`, a new Alembic revision, and
  (by convention) a numbered SQL file. Then run `python -m app.tools.check_schema_parity`.
- In prod, the `migrate` service runs `alembic upgrade head` on every stack update.
- One-off post-Ship-4 Qdrant rename: `deploy/migrate_qdrant_collections.py` (gitignored, see §13).

---

## 12. Testing and tooling

```powershell
.\.venv\Scripts\python.exe -m pytest tests -q
```

```powershell
.\.venv\Scripts\python.exe -m ruff check app
```

- **Status at handover (2026-10-01, this branch): 1618 passed, 1 skipped, about 3m47s** on the
  dev venv. The only warnings are FastAPI `regex=` → `pattern=` deprecations in
  `routes/ingest.py` and `routes/documents.py`.
- Unit and route tests are hermetic. Use `conftest.py` fixtures and avoid live services. `asyncio_mode=auto`.
  Some linkage tests read reference workbooks from local (gitignored) data and skip when those files are absent.
- Coverage clusters: access control (`test_project_access`, `test_org_access`), credits
  (`test_credits`, `test_credit_renewal`, `test_chat_credit_release`), chat contracts
  (`test_chat_pipeline_contracts`, `test_synth_tool_results`, `test_guard_image_routing`),
  extraction (`test_preprocessing`, `test_text_repair`, `test_markdown_normalizer`,
  `test_extraction_guard`, `test_graph_empty_page_gate`), knowledge (`test_knowledge_*`),
  benchmark (`test_time_depth`, `test_composite`, `test_delta`, `test_aftershock`,
  `test_attention`, …), DDR (`test_ddr_*`), linkage (`test_linkage_*`, `test_npt_tables`,
  `test_narrative`), reports (`test_report_*`).
- `tests/e2e/` is empty apart from caches. There are no end-to-end tests in the repo.
- Live checks: `app.tools.api_smoke` (API), `app.tools.probe_chat` (chat quality, spends tokens).
- Module size budget: `python -m app.tools.check_file_length` (400 lines). This is why large modules are
  split into `_private` siblings.
- Local LangGraph debugging: `langgraph dev` (see `langgraph.json`) or the
  `langgraph-dev` service in the local compose file.

---

## 13. Known risks, tech debt and doc drift

Ordered by priority. 🔴 = act before or at handover.

| # | Item | Detail | Suggested action |
|---|---|---|---|
| 🔴1 | **Secrets committed in compose files** | `docker-compose.yml` and `docker-compose.dev.yml` contain literal values for `GEMINI_API_KEY`, `JWT_SECRET`, `RESEND_API_KEY`, `APPLICATIONINSIGHTS_CONNECTION_STRING`, `GOOGLE_CLIENT_SECRET` (prod) and the Postgres password. They are in git history. | Rotate every one. Move them to Portainer env or a secret store and reference them as `${VAR}`. Consider scrubbing history. |
| 🔴2 | `super_admin_password` has a non-empty default in `app/config/_platform.py` | If env does not override it, the bootstrap account uses a known password | Remove the default (require env), rotate the prod super_admin password |
| 🔴3 | **No `celery beat` deployed** | `beat_schedule` defines `purge-stale-demo-daily` and `sweep-credit-holds-hourly`, but no compose file runs beat. Orphaned credit holds are never released, and stale demo accounts are never purged | Add a `celery-beat` service, or a cron job running the CLI equivalents |
| 🔴4 | `deploy/` and `docs/` are **gitignored** | The README, wiki and code reference `deploy/migrate_qdrant_collections.py`, `deploy/renew-ssl.sh`, `deploy/dev-autopull.sh`, `docs/benchmarking/*`, `docs/blog_api_contract.md`. A fresh clone has none of them | Commit them (after checking for secrets) or move them to a shared drive and update the links |
| 🟠5 | One worker per document | Bulk uploads fan out per document, so very large simultaneous uploads risk OOM. Mitigated by per-child memory caps | See the plan in the project memory notes. Page-level fan-out was deferred |
| 🟠6 | Pricing cliff | `gemini-3.7-flash` (linkage) introductory pricing ends **2026-12-31**, then doubles | Update `app/utils/pricing.py` rates. Re-check `COST_ANALYSIS.md` |
| 🟠7 | Paid-tier enforcement off | `billing_enforcement_enabled=False`. Paid orgs are metered, never blocked (soft cap) | Product decision before real billing |
| 🟠8 | Well identity | `well_registry_enabled=False`. `npt_events`/lessons have NULL `well_id`. Cross-document per-well rollups are limited | Needs a structural well-name parser and external IDs |
| 🟡9 | OCR fallback is dead code in prod | `liteparse_ocr_enabled=False` and no tesseract in the image | Either install and enable it, or delete the branch |
| 🟡10 | `table_header_detached` signal is unused | Computed in markdown normalization, but nothing routes on it | Route those pages to VISION, or remove the signal |
| 🟡11 | FR-15 `query_facts` chat tool not implemented | Only a config flag exists, and `app/db/queries/benchmark_facts.py` (named in wiki 21) does not exist | Implement it or remove the flag and the doc reference |
| 🟡12 | Knowledge follow-up loop "on probation" | Delete it if `followup`-origin questions do not produce applied corrections | Review `usage_events.metadata` origin breakdowns |
| 🟡13 | Linkage accuracy is measured on 2 wells | Held-out n=9. The confidence intervals are about 50pp wide | Add wells (about 8 for ±5pp) before quoting accuracy |
| 🟡14 | No E2E tests, no CI test stage | The pipelines build and push images but do not run pytest or ruff | Add a test stage to both pipelines |
| 🟡15 | Local branch hygiene | Local `main` and `dev` are stale versus origin. `origin/dev` was last bumped 2026-07-30 and is about 90 commits behind this branch | Agree whether `dev` is still the staging flow, then sync or retire it |

**Doc drift found while writing this** (the code is correct, the doc is out of date):

- `README.md` says the Alembic head is `010`. It is `029`. *(Fixed in this change.)* Its
  "Project layout" also omits `domain/`, `linkage/`, `agents/knowledge/`, `config/` (now a
  package), `tools/` and `welli/`. Use §4–§5 of this file instead.
- Several `CLAUDE.md` files still name `config.py`, `db/models.py`, `auth/email.py` and
  `db/queries/credits.py` as single files. All four are now packages (`config/`, `db/models/`,
  `auth/email/`, `db/queries/credits/`).
- `app/CLAUDE.md` says `.env` is committed. It is not tracked. The secrets risk is in the compose files (item 1).
- `app/linkage/CLAUDE.md` says `linkage_enabled` defaults to `False`. It is `True`.
- `app/workers/tasks/CLAUDE.md` omits `knowledge.py` and `linkage.py`. See §5.14.
- `wiki/21` "Status: Pre-implementation" is stale. Composite, delta, CPR, aftershock,
  effectiveness, attention, cache and template discovery are all implemented.
- The `app/auth/__init__.py` docstring still mentions Supabase. The system is fully self-hosted.

---

## 14. Handover checklist

**Access to transfer** (verify each one works for the new owner):

- [ ] Azure DevOps org `AIGENITY`, project `WellsynthAI` (repo, pipelines, the `dsaishared` service connection)
- [ ] Azure Container Registry `dsaishared.azurecr.io`
- [ ] Azure VM(s) hosting prod and dev (SSH, `/home/azureuser/wellsynth-data`, `/opt/wellsynthai`), plus Portainer (stack #22)
- [ ] Azure Application Insights resource
- [ ] Google AI Studio / Gemini API project (key, quota, billing)
- [ ] Google Cloud OAuth client (redirect URIs must match `cors_allow_origins`)
- [ ] Resend account (verified domain `wellsynth.ai`)
- [ ] GoDaddy DNS for `wellsynth.ai` (`api`, `api-dev`, apex, `www`, `app`)
- [ ] Let's Encrypt account email (expiry notices)
- [ ] Prod super_admin credentials (rotate them on transfer)
- [ ] Support inbox address (`SUPPORT_INBOX_EMAIL`, currently a personal address in the prod compose file)

**Artifacts to hand over:**

- [ ] `deploy/` and `docs/` directories (gitignored, see §13 item 4)
- [ ] `data/plan_documentations/` (historical plans the wiki links to)
- [ ] Reference corpora used for measurements (Sajaa 37 / Sajaa 1 DDR workbooks and EOWRs, GSWA WCR, etc.). The tests use fixtures, but the measurements in the `CLAUDE.md` files came from these files

**Suggested first week for the new owner:**

1. Bring up the local stack, ingest a small PDF, chat about it, and generate a report.
2. Read `app/CLAUDE.md` and the `CLAUDE.md` files for `auth`, `db/queries`, `pipeline` and `agents/rag`.
3. Run the test suite and ruff. Then run `api_smoke` against dev.
4. Close 🔴 items 1–4 in §13.
5. Walk through one prod deploy end to end (pipeline → Portainer → `migrate` → smoke).

---

## 15. Glossary

| Term | Meaning |
|---|---|
| **DDR** | Daily Drilling Report: a per-day form with activities, hours and depths |
| **EOWR** | End of Well Report: the narrative and tabular summary of a well |
| **WITSML** | XML standard for wellsite data. Parsed deterministically in `domain/ddr.py` |
| **NPT** | Non-Productive Time. A closed IADC-style category vocabulary in `domain/npt.py` |
| **ILT** | Invisible Lost Time: time lost relative to the best composite that is not booked as NPT |
| **Best composite** | The best achieved time per (hole section × depth band × activity) across offset wells (FR-3) |
| **CPR** | Composite Performance Ratio = actual time ÷ best composite time (FR-8) |
| **BHA** | Bottom Hole Assembly. **Leg**: a lateral or sidetrack branch of the well |
| **Layer A / Layer B** | Raw per-document insight JSON (`document_insights`) and normalized facts (`npt_events`, `lessons`) |
| **Document Insight Layer** | Post-ingest knowledge cards with quote-verified findings and corrections |
| **`artifact_class`** | Document governance class (e.g. `master`, `program`, `learning`, `other`) that drives retrieval tier weights |
| **Lane** | Ingestion mode: `realtime` (online Gemini) or `batch` (Gemini Batch API) |
| **Hold / reserve / settle** | Credit lifecycle: provisional debit → finalize against measured tokens (or release) |
| **Ship N** | Internal milestone numbering for the multi-tenant and billing work (Ships 1–9, see README) |
| **FR-N** | Benchmarking functional requirements (FR-1 … FR-16, see `wiki/21`) |
