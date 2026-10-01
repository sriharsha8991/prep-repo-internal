# Wellsynthai Low-Level Design

## 1. Purpose

This document explains the current Wellsynthai backend application at low-level design depth. It describes the running services, request lifecycle, module responsibilities, data model, ingestion pipeline, retrieval and chat flow, report generation, support workflows, billing controls, persistence, storage, vector indexing, observability, and operational behavior.

The design below is based on the current codebase. The current implementation uses FastAPI, Celery, Redis, local Postgres, local filesystem storage, Qdrant, and Gemini. Some older project notes still mention Supabase; the current application code and wiki describe the active migration state as self-hosted local Postgres, self-hosted JWT auth, and local filesystem buckets.

## 2. System Overview

Wellsynthai is a document intelligence and RAG application for well-operations documents. A user uploads PDFs into projects. The backend extracts page-level markdown, tables, images, and entities; stores source/artifact files; indexes chunks/entities/images in Qdrant; and exposes search, chat, reports, document viewing, support, memories, billing, and admin endpoints.

Core product loop:

```text
User uploads PDF
  -> ingest creates document/job records
  -> worker extracts per-page content and artifacts
  -> postprocessor assembles markdown/entities/tables
  -> vectorizer indexes chunks/entities/images in Qdrant
  -> chat/search/report retrieve scoped evidence
  -> LLM answers with citations and source metadata
```

## 3. Runtime Topology

The application is deployed as a compose stack. The development compose file uses the same service shape with dev-specific names, ports, volumes, and image tags.

| Service | Responsibility |
| --- | --- |
| FastAPI app | HTTP API, auth middleware, request routing, singleton Gemini/embedder/Qdrant clients, health/readiness. |
| Celery worker | Realtime ingestion, report generation, memory extraction, cleanup tasks. |
| Celery batch worker | Gemini Batch ingestion polling and collection queues. |
| Postgres | System of record for users, profiles, organizations, projects, documents, jobs, chat rows, reports, usage, support, credits, feedback, memories. |
| Redis | Celery broker, Celery result backend, rate-limit/session/state cache. |
| Qdrant | Vector store for document chunks, entities, and images. |
| Local filesystem storage | Bucket-like storage for source PDFs, rendered pages, extracted artifacts, support attachments. |
| Gemini | Multimodal extraction, classification, chat guard/synthesis, embeddings, report synthesis. |
| Resend | Transactional email for auth/support flows. |

The runtime dependencies are intentionally narrow: app code owns auth, data access, storage paths, and tenant scoping; external services are used for AI calls, vector search, email, and infrastructure primitives.

## 4. Application Entry Point

The FastAPI application is defined in `app/main.py`.

Startup lifecycle:

1. Load `Settings` from environment via `app/config.py`.
2. Configure structured logging and telemetry.
3. Ensure upload/storage directories exist.
4. Create process-wide singletons:
   - `GeminiClient`
   - `GeminiEmbedder`
   - `QdrantVectorStore`
5. Ensure Qdrant collections exist.
6. Reconcile stale strategic reports left in queued/running states after a crash.
7. Seed super admin when configured.
8. Sync free-tier token budget into the database.

Shutdown lifecycle:

1. Close Qdrant client.
2. Dispose SQLAlchemy async engine.
3. Emit app stopped log.

Middleware order:

1. `GZipMiddleware` compresses large text/JSON responses.
2. `CORSMiddleware` enforces configured origin allow-list.
3. `AuthMiddleware` verifies bearer JWTs on protected paths.
4. `RequestLoggingMiddleware` records request metadata when enabled.

The app includes routers for auth, users/admin, usage/admin, projects, documents, ingest, search, chat, chat sessions, memories, support, feedback, demo, reports, report templates, and Welli.

## 5. Configuration Design

All runtime settings live in `app/config.py` as a Pydantic `Settings` class. Configuration is environment-driven and covers:

- Gemini API key, model names, timeouts, token ceilings, safety behavior, adaptive pacing.
- Ingestion concurrency, routing thresholds, rendering DPI, progress-write interval.
- Batch ingestion lane settings.
- Billing/free-tier token budgets and hold TTLs.
- Demo signup/rate-limit settings.
- Chat model selection and RAG behavior.
- Report generation model selection and concurrency.
- Postgres, Redis, Qdrant, storage, CORS, JWT, telemetry, and support settings.

Important concurrency defaults:

- `max_concurrent_pages = 60` per process for Gemini-bound page extraction.
- `render_concurrency = 8` for PyMuPDF rasterization threads.
- Celery worker concurrency is configured at process level in compose.
- Batch poll/collect queues are isolated from realtime extraction queues.

## 6. Authentication and Authorization

Auth is self-hosted in the backend.

Primary modules:

- `app/auth/routes.py`: login, refresh, change password, demo signup, verification, current profile endpoints.
- `app/auth/middleware.py`: bearer-token verification and request user loading.
- `app/auth/jwt_verify.py`: token creation and validation.
- `app/auth/password.py`: password hashing/verification.
- `app/auth/deps.py`: route dependencies for current user and role enforcement.
- `app/auth/project_access.py`: project-level access checks.

Credential flow:

1. User signs in through `POST /auth/login` with email and password.
2. Backend verifies password hash from `users`.
3. Backend returns access and refresh JWTs.
4. Protected requests carry `Authorization: Bearer <access_token>`.
5. `AuthMiddleware` verifies the token and loads `users_profile`, organization, tier, demo flag, and email verification state.
6. Routes access the loaded `AuthUser` through `Depends(current_user)`.

Anonymous routes are explicitly listed in `AuthMiddleware`, including health, docs, login/refresh, demo signup/verification, Welli chat, and demo request endpoints.

Authorization model:

- Roles are defined by `ROLE_VALUES` in `app/db/models.py`.
- Project ownership and shared-project access are enforced in app code.
- Admin and super-admin behavior is explicit; there is no silent global bypass for normal list/read paths.
- Project/doc access checks are performed before reading or mutating scoped records.
- Vector search is tenant-scoped through project-id/user-id filters and org-specific collection views.

## 7. Persistence Layer

SQLAlchemy async engine lives in `app/db/engine.py`.

Design decisions:

- `DATABASE_URL` and `WORKER_DATABASE_URL` are normalized to `postgresql+asyncpg://`.
- A process-level async engine/session factory is lazily created.
- Engine pool is sized for direct Postgres, not pgbouncer.
- Celery tasks create a fresh event loop and dispose the engine before closing the loop.

The ORM models are in `app/db/models.py`. Migrations are in `migrations/` and Alembic files are in `alembic/`.

Core tables:

| Table | Purpose |
| --- | --- |
| `users` | Credential rows, password hash, email confirmation, verification token metadata. |
| `users_profile` | Role, display name, organization, memory toggle, invite metadata, signup source. |
| `organizations` | Tenant identity, frozen slug, tier, org credit pool. |
| `projects` | User-owned or org-shared document containers. |
| `documents` | Stable document identity, filename, project, owner, org, artifact class, confidentiality, content hash. |
| `jobs` | Ingestion run status, phase, page counts, lane, batch metadata. |
| `page_extractions` | Shared content-hash/page extraction cache for deduplication. |
| `usage_events` | Token/cost ledger for ingest/chat/report operations. |
| `chat_sessions` | Durable chat sidebar/index rows. |
| `chat_messages` | Persistent transcript rows for non-temp chats. |
| `user_memories` | Long-term user/project-scoped memories. |
| `support_tickets` | User support tickets. |
| `support_ticket_attachments` | Attachment metadata and storage keys. |
| `support_ticket_notes` | Admin notes/status history. |
| `strategic_reports` | Report generation state, markdown result, section metadata, agentic blackboard. |
| `report_templates` | User-defined/custom report templates. |
| `subscription_plans` | Plan configuration and monthly token grants. |
| `user_subscriptions` | Per-user plan/balance/feature usage/demo state. |
| `credit_ledger` | Append-only credit/token movement audit. |
| `credit_holds` | Reservation lifecycle for token budget enforcement. |
| `user_feedback` | Feedback submissions and bonus token awards. |
| `demo_requests` | Public demo interest submissions. |

Important identity rules:

- `documents.id` is stable and durable.
- `jobs.id` is per ingestion run.
- `filename` is display data.
- `content_hash` allows shared extraction cache reuse.
- `organization.slug` is a frozen physical Qdrant collection key.

## 8. Local Storage Design

Storage is implemented by `app/storage/local_storage.py` and behaves like bucket storage on a local filesystem.

Storage root contains three logical buckets:

```text
<storage_root>/pdfs/<user_id>/<project_id>/<document_id>/source.pdf
<storage_root>/pages/<user_id>/<project_id>/<document_id>/page_NNN.png
<storage_root>/artifacts/<user_id>/<project_id>/<document_id>/...
```

Artifacts include:

- `full_document.md`
- per-page markdown: `page_NNN.md`
- per-page resume state: `page_NNN.state.json`
- table JSON and markdown files
- `entities.json`
- support ticket attachments under support-specific keys

Storage design goals:

- Keep source PDF bytes out of database.
- Let document viewer serve PDFs/page images efficiently.
- Persist per-page state as each page completes, enabling ingestion resume.
- Store postprocessed markdown/tables/entities close to document identity.
- Delete project/document storage before DB cascade when explicit cleanup is needed.

## 9. Vector Store Design

Qdrant access is implemented in `app/vectorstore/qdrant_store.py`.

There are three logical vector collections per tenant scope:

| Collection type | Default collection | Purpose |
| --- | --- | --- |
| Chunks | `wellsynthai_documents` | Narrative/text/table chunks. |
| Entities | `wellsynthai_entities` | Normalized entities such as formations, units, wells, equipment. |
| Images | `wellsynthai_images` | Image descriptions and page/figure evidence. |

For organization-scoped users, collection names are derived from frozen org slug:

```text
wellsynthai_<org_slug>_documents
wellsynthai_<org_slug>_entities
wellsynthai_<org_slug>_images
```

Each payload includes tenant and source metadata:

- `user_id`
- `role`
- `organization`
- `project_id`
- `document_id`
- `doc_name`
- `job_id`
- `filename`
- `page_num`
- type-specific fields such as `chunk_index`, `kind`, `table_id`, `bbox`, `entity_type`, `canonical_id`, `unit`, `image_type`, `description`.

Search scoping is mandatory. The store builds filters using either:

- `project_ids` for shared-project-aware retrieval; or
- `user_id` as legacy fallback.

An empty project-id list is rejected to avoid unfiltered searches.

## 10. API Surface

### System

- `GET /`: service metadata and capability summary.
- `GET /health`: liveness.
- `GET /ready`: dependency readiness.
- `GET /health/deep`: deep readiness alias.

### Auth and Me

- `POST /auth/login`: email/password login.
- `POST /auth/refresh`: refresh token rotation.
- `POST /auth/change-password`: authenticated password change.
- `GET /auth/me`: current profile.
- `PATCH /auth/me/profile`: own profile update.
- Demo signup/verification endpoints for public demo users.

### Projects

- `POST /projects`: create a project.
- `GET /projects`: list personal and accessible shared projects.
- `GET /projects/{project_id}`: load a project.
- `PATCH /projects/{project_id}`: update metadata.
- `DELETE /projects/{project_id}`: delete project, storage, vectors, and DB records.

### Documents

- `GET /documents`: list accessible documents with latest job inline.
- `GET /documents/{document_id}`: document detail.
- `PATCH /documents/{document_id}`: governance metadata.
- `DELETE /documents/{document_id}`: delete storage, vectors, DB row, extraction cache when orphaned.
- Document artifact endpoints serve source PDF, full markdown, page markdown, page images, and crops.

### Ingest

- `POST /ingest`: ingest a server-side PDF path.
- `POST /ingest/upload`: upload and ingest a PDF.
- Batch upload variants dispatch multiple files.
- `GET /ingest`: list jobs.
- `GET /ingest/{job_id}`: job status and progress.

### Search and Chat

- `GET /search`: direct semantic search over indexed evidence.
- `POST /chat`: non-streaming RAG chat.
- Streaming chat endpoint returns Server-Sent Events.
- Chat session endpoints manage sidebar/list/rename/delete behavior.

### Reports

- `POST /projects/{project_id}/reports`: dispatch report generation.
- `GET /projects/{project_id}/reports`: list reports.
- `GET /projects/{project_id}/reports/{report_id}`: report detail.
- `GET /projects/{project_id}/reports/{report_id}/pdf`: render markdown to PDF on demand.
- `DELETE /projects/{project_id}/reports/{report_id}`: delete report row.
- Report template endpoints manage built-in and custom templates.

### Support, Feedback, Memories, Admin

- Support ticket endpoints create/list/detail tickets and attachments.
- Feedback endpoint records ratings/messages and can grant token bonuses.
- Memory endpoints list/forget/toggle long-term memory.
- Admin endpoints manage users, usage, projects, credits, and support triage.

## 11. Ingestion Upload Flow

Primary module: `app/routes/ingest.py`.

Upload flow:

1. Request authenticates through `AuthMiddleware`.
2. Route validates project access with `load_and_assert`.
3. Route validates artifact class, confidentiality, lane, filename, size, and PDF extension.
4. Route computes SHA-256 content hash.
5. Route checks for duplicate document in the same project when dedup is enabled.
6. Route creates `documents` and `jobs` rows.
7. Route writes `source.pdf` into local storage.
8. Route reserves ingest credits using estimated tokens.
9. Route dispatches Celery task:
   - `ingest_document` on queue `extract` for realtime lane.
   - `ingest_batch_submit` on queue `rasterize` for batch lane.
10. Client polls `GET /ingest/{job_id}` for status/progress.

Document/job separation:

- A document is durable and can survive re-ingestions.
- A job represents one processing attempt.
- Re-ingestion after failure can create a new job for the existing document.

## 12. Realtime Ingestion Worker Flow

Primary modules:

- `app/workers/tasks/ingest.py`
- `app/pipeline/orchestrator.py`
- `app/pipeline/preprocessing.py`
- `app/pipeline/page_streamer.py`
- `app/pipeline/page_router.py`
- `app/pipeline/postprocessor.py`
- `app/agents/graph.py`
- `app/agents/extractor.py`
- `app/agents/escalation.py`

Celery task flow:

1. Task updates job row to `running` and `starting`.
2. Task creates runtime dependencies: `GeminiClient`, `QdrantVectorStore`, `LocalStorage`, `PipelineOrchestrator`.
3. Task resolves source PDF from local storage.
4. Orchestrator runs the pipeline.
5. Task writes final job state as `succeeded` or `failed`.
6. On success, task records usage and settles credit hold.
7. On failure, task releases credit hold.

The task runs async code inside a fresh event loop. It cancels pending tasks, disposes the DB engine, shuts down async generators, and closes the event loop at the end.

## 13. Ingestion Pipeline Phases

### Phase 0: Fast Preprocessing

`Preprocessor.extract_fast` uses PyMuPDF to read page-level metadata and text:

- raw text
- character count
- image count
- page dimensions
- scanned-page heuristic

The orchestrator writes initial `pages_total` and starts a progress heartbeat that flushes current phase and page progress to `jobs` every few seconds.

### Phase 0b: Background Enrichment

`Preprocessor.enrich_baselines` runs concurrently with page extraction. It enriches baseline text with layout-aware markdown. The orchestrator awaits it before postprocessing, because fallback text and full-document assembly rely on enriched baselines.

### Phase 1: Streaming Rasterization

`rasterize_pages` renders PDF pages to PNG using bounded threads and pushes `PageImage` objects into a `PageDeque`.

Behavior:

- Adaptive DPI can render scanned/dense pages at higher resolution and sparse pages lower.
- Page PNGs are uploaded to local `pages` storage as they are produced.
- Table detection can influence routing before spending LLM calls.
- Workers are started before rasterization, so extraction begins as soon as the first page is ready.

### Phase 2: Page Routing and Extraction

Each worker pops pages from `PageDeque` and chooses a route.

Routes:

| Route | Trigger | Behavior |
| --- | --- | --- |
| `LITEPARSE` | Text-rich, no substantial tables/images | Use enriched/baseline text, no Gemini extraction call. |
| `CLASSIFY` | Ambiguous sparse text | Use cheap classifier to decide liteparse vs VISION. |
| `VISION` | Scanned, image-heavy, table-heavy, or classifier-approved | Run multimodal LangGraph extraction. |

Resume behavior:

- Before extracting, the worker checks for existing `page_NNN.state.json`.
- If a clean final state exists and is not degraded, it is reused.
- Error/degraded pages are retried instead of reused.

VISION behavior:

1. Page image becomes LangGraph state.
2. Extractor runs Gemini multimodal extraction.
3. Extractor self-validates by returning confidence and gaps.
4. Numeric guardrails and entity normalization can add gaps or force escalation.
5. If validation passes, extracted state becomes final.
6. If validation fails, escalation model re-extracts with gaps/prior JSON and produces final state.
7. Worker persists final state and optionally saves it into shared content-hash cache.

Error behavior:

- Per-page failures do not cancel the whole document.
- Failed pages fall back to baseline text.
- Such pages are marked `degraded` and counted in job summary/quality logs.

### Phase 3: Postprocessing

`PostProcessor` converts per-page final states into document artifacts.

Responsibilities:

- Choose final extracted markdown or fallback markdown for each page.
- Write per-page markdown files.
- Assemble `full_document.md`.
- Merge cross-page tables where markers indicate continuation.
- Write table JSON/markdown artifacts.
- Normalize and resolve entities.
- Write `entities.json`.

### Phase 4: Vectorization

Vectorization is shared by realtime and batch lanes through `PipelineOrchestrator.finalize`.

Steps:

1. Resolve org-scoped Qdrant collection view.
2. Ensure tenant collections exist.
3. Build a page map from final page results.
4. Gather persisted page PNG paths.
5. Chunk page markdown using `DocumentChunker`.
6. Delete existing vectors for the document.
7. Embed chunks with `GeminiEmbedder`.
8. Upsert chunks to Qdrant.
9. Build and upsert entity points.
10. Build and upsert image description points.
11. Track approximate embedding token usage.

Idempotency rule: old vectors are deleted before new upsert, so re-ingestion does not leave stale points.

### Phase 5: Artifact Auto-Classification

If the document's `artifact_class` is still `other`, the orchestrator samples assembled markdown and headings and calls the artifact classifier. A confident non-`other` result updates the document row. User-provided artifact classes are respected and not overwritten.

## 14. Batch Ingestion Lane

Batch lane uses Gemini Batch API for lower cost and asynchronous completion.

Task split:

- `ingest_batch_submit`: rasterizes and submits inline batch jobs, writes batch metadata, schedules polling.
- `poll_batch_job`: periodically checks Gemini batch state.
- `collect_batch_job`: collects completed batch results and finalizes the document.

Differences from realtime:

- No per-page escalation loop.
- Completion can take much longer.
- Poll and collect queues are served by a dedicated worker so long finalization does not block realtime extraction.
- Final postprocessing/vectorization path is shared with realtime through `finalize`.

## 15. Retrieval and Search Design

Direct search is implemented in `app/routes/search.py` and vector search helpers.

Flow:

1. Authenticate user.
2. Reserve/gate search credits.
3. Resolve process-wide embedder and Qdrant store from app state.
4. Bind store to user's org collection view.
5. Resolve accessible project ids.
6. Validate requested project scope when `project_id` is provided.
7. Embed query.
8. Search Qdrant with project/user/document/page filters.
9. Settle credits with effectively zero measured token cost for query search.
10. Return scored payloads.

The key security property is that retrieval scope is resolved before Qdrant search and encoded into the filter. Empty accessible-project sets return no results.

## 16. Chat/RAG Design

Primary modules:

- `app/routes/chat.py`
- `app/agents/rag/agent.py`
- `app/agents/rag/guard.py`
- `app/agents/rag/synthesizer.py`
- `app/agents/rag/retriever.py`
- `app/agents/rag/evidence.py`
- `app/agents/rag/stm.py`
- `app/agents/rag/sessions.py`

Chat request body includes:

- `message`
- optional `session_id`
- optional `project_id`
- optional `document_id`
- `temp` flag for incognito/short-lived sessions

Route flow:

1. Authenticate user.
2. Enforce demo IP/rate limits when applicable.
3. Reserve chat credits.
4. Resolve embedder and Qdrant store from app state.
5. Bind Qdrant store to user's org collection view.
6. Validate project/document scope and compute accessible project ids.
7. Create or reuse session id.
8. Handle slash commands before RAG when applicable.
9. Build context card and short-circuit empty corpus.
10. Run non-streaming or streaming RAG agent.
11. Persist chat session/message rows for non-temp chats.
12. Settle/release credits based on measured token metadata.

Agent architecture:

```text
Layer 0: Regex prefilter
  -> handles greetings/thanks and other zero-cost replies

Layer 1: Guard model
  -> decides reply, clarify, or retrieve
  -> can answer app-help/account/catalog style questions without RAG

Layer 2: Synthesizer model
  -> function calls retrieval tools
  -> answers from evidence with citations
```

Retrieval tool behavior:

- `rag_search` embeds the query once.
- Searches chunks and entities in parallel.
- Searches images only when the user explicitly asks for visual content.
- Dedupes and sorts results.
- Expands neighboring narrative chunks around top hits.
- Returns compact evidence to the synthesizer.

Synthesizer behavior:

- Uses Gemini with function calling.
- Has a small tool-call budget.
- Can call `rag_search`, `fetch_document_outline`, or `fetch_table`.
- Builds source/evidence structures for the API response.
- Can attach page images for multimodal verification when requested.

Session/memory behavior:

- Short-term memory/session state is stored through the RAG session layer, with temp chats using a TTL.
- Durable chat sidebar and transcript rows are written to Postgres for non-temp chats.
- After answered turns, a Celery memory task can extract long-term memories unless temp mode is used.

## 17. Reports Design

Primary modules:

- `app/routes/reports.py`
- `app/workers/tasks/report.py`
- `app/agents/reports/orchestrator.py`
- `app/agents/reports/builder.py`
- `app/agents/reports/registry.py`
- `app/agents/reports/templates.py`
- `app/agents/reports/pdf_renderer.py`

Report creation flow:

1. User calls `POST /projects/{project_id}/reports`.
2. Route validates project access.
3. Route verifies project has documents.
4. Route enforces one in-flight report per project.
5. Route resolves and validates template.
6. Route reserves report credits.
7. Route inserts `strategic_reports` row with `queued` status.
8. Route dispatches `report.generate` Celery task on `reports` queue.
9. Route stores Celery task id in the report row.

Worker flow:

1. Create Gemini client, embedder, Qdrant store.
2. Resolve project's organization slug.
3. Bind vector store to project org collection.
4. Run `run_report_generation` in a fresh event loop.
5. Close Qdrant and dispose DB engine at the end.

Report orchestrator flow:

1. Resolve template.
2. Mark report `running`.
3. List project documents.
4. Compute deterministic project metrics where possible.
5. Generate sections using legacy single-shot or agentic master path depending on config.
6. Assemble markdown with section results, coverage appendix, and data gap register.
7. Store markdown, section metadata, status, timings, and errors in `strategic_reports`.
8. Usage/credit settlement is handled by report generation support code.

PDF export renders stored report markdown on demand and does not need to persist a generated PDF artifact.

## 18. Projects and Documents Design

Projects are containers for documents. A project has an owner and can be personal or shared within a paid organization.

Project deletion:

1. Validate delete access.
2. Resolve project owner because storage is owner-keyed.
3. Capture document content hashes.
4. Delete local storage subtree.
5. Delete Qdrant points by project.
6. Delete project row; DB cascades dependent documents/jobs.
7. Purge orphaned page-extraction cache entries.

Document deletion:

1. Resolve document and project access.
2. Require uploader/admin/super_admin to delete.
3. Delete local storage subtree.
4. Delete Qdrant points by document.
5. Delete document row.
6. Purge orphaned page-extraction cache entries when no remaining document uses the hash.

Document serving:

- Source PDFs are served from local `pdfs` storage and can support range-based viewer access.
- Page images are served from local `pages` storage.
- Page markdown, full markdown, table artifacts, entity JSON, and crops are built from local artifacts and source PDFs.

## 19. Support Ticket Design

Primary module: `app/routes/support.py`.

Ticket submit flow:

1. Authenticate user.
2. Enforce support submission rate limits.
3. Validate category, subject, message, diagnostics JSON, attachment count, MIME type, and size.
4. Insert support ticket row.
5. Write attachments to local artifact storage.
6. Insert attachment metadata rows.
7. Commit DB transaction.
8. Send support email with ticket metadata.
9. Mark email sent and store email message id.

Attachment design:

- Allowed MIME types are limited to PNG, JPEG, PDF, and text.
- Filenames are sanitized.
- Attachment object names are prefixed with a short UUID to avoid collisions.
- The DB stores original filename, MIME type, size, and storage key.

Admin support routes handle triage/status/note operations separately.

## 20. Memory Design

Memory endpoints live in `app/routes/memories.py`.

Memory scopes:

- `preference`
- `fact`
- `project_pin`
- `style`
- `dismissed`

Memory behavior:

- Memories are owner-scoped by user id.
- `project_id = NULL` means global personal memory.
- Non-null `project_id` means project-isolated memory.
- Users can toggle cross-session memory via `users_profile.memory_enabled`.
- Users can list memories, forget one memory, or forget all memories in a scope.
- Temp/incognito chat skips long-term memory capture.

## 21. Billing, Credits, Demo, and Usage

Billing is token-budget based.

Core ideas:

- Balances represent raw tokens.
- Free-tier enforcement can be enabled independently of global billing enforcement.
- Resource-consuming operations reserve estimated tokens before work starts.
- Completion settles holds against measured tokens/cost.
- Failure releases holds.
- A periodic cleanup task releases stale open holds after TTL.

Metered features:

- `chat`
- `ingest`
- `report`
- `search`

Important tables:

- `subscription_plans`
- `user_subscriptions`
- `organizations`
- `credit_ledger`
- `credit_holds`
- `usage_events`
- `user_feedback`

Demo-specific behavior:

- Public demo self-signup creates a free user with demo marker.
- Demo accounts can get immediate access while verification runs in background.
- Demo IP throttling prevents farming.
- Unverified demo accounts can be purged by cleanup task.
- Feedback can grant a one-time bonus when quality threshold is met.

## 22. Celery Design

Celery app is defined in `app/workers/celery_app.py`.

Queues:

| Queue | Purpose |
| --- | --- |
| `extract` | Realtime ingestion task. |
| `rasterize` | Batch-lane submit/rasterize task. |
| `aggregate` | Reserved/document-level aggregation queue. |
| `reports` | Strategic report generation. |
| `batch_poll` | Lightweight batch polling. |
| `batch_collect` | Heavy batch result collection/finalization. |
| `default` | Cleanup and miscellaneous tasks. |

Reliability settings:

- JSON serialization only.
- UTC timestamps.
- `task_acks_late=True`.
- `task_reject_on_worker_lost=True`.
- `worker_prefetch_multiplier=1`.
- `worker_max_tasks_per_child=5`.

Periodic tasks, when Celery beat or equivalent cron is running:

- Purge stale demo users daily.
- Sweep stale credit holds hourly.

Task implementation pattern:

1. Create fresh asyncio event loop per task.
2. Run async orchestrator or async helper.
3. Cancel pending tasks in `finally`.
4. Close external clients.
5. Dispose SQLAlchemy engine.
6. Shutdown async generators and close loop.

## 23. Observability Design

Observability modules live under `app/observability/`.

Responsibilities:

- Readiness checks for dependent services.
- Request logging middleware.
- Telemetry configuration.
- Celery instrumentation.
- Metrics counters/histograms.
- Spans around major operations.
- Operation/user context binding.

Common span/metric domains:

- `auth.load_profile`
- `chat.resolve_scope`
- `rag.retrieve`
- `ingest.preprocess.fast`
- `ingest.rasterize`
- `ingest.postprocess`
- `ingest.vectorize`
- `report.*`

Useful structured log events include:

- `app_started`, `app_stopped`
- `pipeline_preprocessing_fast_done`
- `worker_page_complete`
- `worker_page_failed`
- `pipeline_extraction_done`
- `pipeline_vectorize_done`
- `pipeline_degraded_pages_high`
- `pipeline_complete`
- `pipeline_failed`
- `agent_guard`
- `agent_done`
- `rag_search_done`
- `support_email_send_failed`

## 24. Error Handling and Resilience

Global API behavior:

- `app/main.py` has a global exception handler returning HTTP 500 with error type.
- Route-level validation uses FastAPI/Pydantic and explicit `HTTPException` responses.
- Auth middleware returns structured 401/403 responses before route handlers.

Ingestion resilience:

- Per-page extraction failures fall back to baseline text.
- Degraded pages are tracked and reported.
- Page state JSON enables resume after partial worker failure.
- Content-hash page extraction cache avoids re-paying for identical PDFs.
- Progress heartbeat keeps the job row fresh during long runs.
- Celery retries and late ack settings reduce lost work risk.

Vector resilience:

- Collection creation is idempotent.
- Re-ingestion deletes old document vectors before upserting new ones.
- Search rejects empty scope filters.

Credit resilience:

- Worker-backed holds use `ref_id` idempotency.
- Success settles measured usage.
- Failure releases holds.
- Sweeper handles abandoned open holds.

Report resilience:

- App startup marks stale queued/running reports as failed after restart.
- Section generation can convert per-section exceptions into section-level error placeholders.
- Report worker retries transient 429/503/resource-exhausted errors once.

## 25. Security and Isolation

Security controls:

- Protected routes require bearer JWT.
- Passwords are hashed; plaintext passwords are never stored.
- Anonymous paths are explicit.
- Role checks are dependency-based.
- Project/document access checks happen before scoped operations.
- `scope=all` style admin access requires super-admin where implemented.
- Qdrant retrieval is filtered by accessible projects or user id.
- Local storage paths are constructed from server-side user/project/document ids.
- Upload size and PDF extension are validated.
- Support attachments have MIME/type/count/size restrictions.
- Demo and support endpoints have rate limits.
- LLM outputs are not treated as authorization decisions.

Important caution:

- Environment files and compose files may contain operational secrets in development. Design documentation must not duplicate or expose those values.

## 26. Performance Characteristics

Ingestion:

- One document maps to one Celery task.
- Within a document, per-page work is highly concurrent.
- The rasterizer streams pages to workers instead of waiting for all pages.
- LLM calls are globally gated by `GeminiClient` semaphore and pacing.
- Liteparse routing avoids unnecessary multimodal calls for text-rich pages.
- Batch lane trades latency for cost.

Retrieval/chat:

- Query embedding happens once per RAG search.
- Chunk/entity searches run in parallel.
- Image search runs only when the user asks for visuals.
- Neighbor expansion is bounded to top narrative hits.
- Guard/prefilter avoid heavy synthesis when retrieval is not needed.

Reports:

- Sections can be generated concurrently within configured semaphores.
- Project metrics and coverage appendices are deterministic where possible.
- PDF export is on demand.

## 27. Deployment and Operations

Compose responsibilities:

- Postgres starts with data volume and init SQL on fresh database.
- `migrate` service runs Alembic before app/workers start.
- Qdrant and Redis use persistent volumes.
- FastAPI app exposes HTTP port.
- Workers consume queues from Redis.
- App and workers mount storage paths consistently so source PDFs/artifacts are visible to both.

Operational checks:

- `GET /health` for liveness.
- `GET /ready` for dependency readiness.
- Celery worker logs for ingest/report task failures.
- Postgres `jobs` rows for ingestion status.
- `strategic_reports` rows for report status.
- Qdrant collection existence and payload filters for retrieval issues.
- Local storage subtrees for missing PDF/page/artifact issues.

## 28. Testing Surface

The repository includes tests covering agents, artifact classification, chat sessions, classifier behavior, credits, entity normalization, guardrails, ingest artifact class, memory, observability, page streamer, postprocessor, preprocessing, tier-weighting governance, slash commands, support admin, and support tickets.

High-value test targets by subsystem:

- Auth: JWT validation, role checks, anonymous route bypasses.
- Projects/documents: ownership, shared access, delete cleanup.
- Ingest: route validation, dedup behavior, page routing, postprocessing, vector upsert calls.
- Chat/search: scope filters, empty corpus soft-land, slash command short-circuit, session persistence.
- Reports: one-in-flight guard, template resolution, row status transitions.
- Billing: reserve/settle/release idempotency, stale hold sweeper.
- Support: attachment validation, rate limits, owner-only reads.

## 29. Key Design Invariants

1. Every protected route starts with a verified `AuthUser`.
2. Every project/document operation validates access before touching scoped data.
3. Every vector query is scoped by accessible project ids or user id.
4. `documents.id` is durable; `jobs.id` is per run.
5. Storage paths are keyed by user/project/document ids, not display names.
6. Re-ingestion deletes old document vectors before inserting new vectors.
7. Page-level failures degrade locally and do not abort the whole document unless pipeline-level failure occurs.
8. Credit holds are settled or released exactly once where possible.
9. Batch poll/collect work is isolated from realtime extraction queues.
10. Documentation and design must not copy runtime secrets from environment/compose files.

## 30. Module Ownership Map

```text
app/main.py                 FastAPI app, middleware, routers, lifespan
app/config.py               Environment settings
app/auth/                   Login, JWT, password, auth middleware, roles
app/routes/                 HTTP resource routes
app/db/                     SQLAlchemy engine, ORM models, query helpers
app/storage/                Local filesystem storage and Redis state helpers
app/workers/                Celery app and background tasks
app/pipeline/               PDF ingestion orchestration and transforms
app/agents/graph.py         Per-page extraction LangGraph
app/agents/extractor.py     Multimodal extraction call
app/agents/escalation.py    Escalation extraction call
app/agents/rag/             Chat guard, retriever, synthesizer, evidence, sessions
app/agents/reports/         Strategic report orchestration and rendering
app/vectorstore/            Chunking, embeddings, Qdrant, entity/image indexing
app/retrieval/              Reranking/scoring governance helpers
app/domain/                 Well taxonomy, units, formations, entity resolution
app/billing/                Credit gates, estimates, demo guards
app/observability/          Telemetry, metrics, spans, request logging, readiness
app/utils/                  Gemini client, pricing, logging, rate limits, SSE
```

## 31. End-to-End Flow Summaries

### PDF to Answer

```text
POST /ingest/upload
  -> document/job rows
  -> local source.pdf
  -> Celery ingest task
  -> preprocess + rasterize + route pages
  -> extract/escalate or liteparse
  -> postprocess full markdown/tables/entities
  -> chunk/embed/upsert Qdrant

POST /chat
  -> validate user and scope
  -> guard decides retrieval
  -> rag_search retrieves chunks/entities/images
  -> synthesizer answers with citations
  -> chat rows/usage/credits updated
```

### Project to Report

```text
POST /projects/{project_id}/reports
  -> validate project and template
  -> reserve credits
  -> insert strategic_report row
  -> Celery report.generate
  -> retrieve project evidence
  -> generate sections
  -> assemble markdown + appendices
  -> store report row
  -> optional PDF render on GET /pdf
```

### User Support Ticket

```text
POST /support/tickets
  -> validate rate limit and fields
  -> validate attachments
  -> insert ticket
  -> write attachments
  -> send email
  -> user/admin can track via support routes
```

## 32. Future Design Notes

Areas to watch when extending the system:

- Keep storage and vector paths tied to stable ids, not filenames.
- Avoid adding new ingestion stages unless they directly improve PDF-to-grounded-answer quality.
- Avoid unscoped vector queries; treat them as security bugs.
- Prefer extending existing route/query/helper boundaries over introducing parallel code paths.
- Keep batch/realtime finalization shared to avoid divergent artifact/vector formats.
- Keep report generation grounded in project documents and Qdrant evidence.
- Keep demo/public flows isolated from invited B2B and admin workflows.
