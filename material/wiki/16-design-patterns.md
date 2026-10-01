# 16 — Design Patterns

> _Owns: the recurring software design patterns the backend deliberately
> commits to, where each one lives, and why. Read this to understand the
> shape of the code before adding to it — new code should reuse an existing
> pattern, not invent a parallel one._
> _Added 2026-06-30. Code is the source of truth; this page links to the
> file/line that implements each pattern and explains intent._

## How to use this page

When you add a collaborator, a tool, a storage backend, or a processing
stage, find the matching pattern below and follow it. The guardrails
([.github/copilot-instructions.md](../.github/copilot-instructions.md))
say *prefer deletion over addition* and *no parallel abstractions* — this
page is the catalogue of the abstractions that already exist so you don't
build a second one.

---

## Architectural patterns

### Pipeline / staged processing
The ingestion run is an explicit, ordered set of phases rather than one
monolith. Each phase has a single responsibility and hands a typed result
to the next.

- [app/pipeline/orchestrator.py](../app/pipeline/orchestrator.py) — Phase
  0 baseline → Phase 1 rasterize → Phase 2 extract/escalate → Phase 3
  postprocess → Phase 4 vectorize.
- **Why:** lets each phase be tuned, traced, and memory-bounded
  independently (see [03-ingestion-pipeline.md](03-ingestion-pipeline.md)).
- **Adding a stage:** the guardrails forbid new extraction stages. Fold
  logic into an existing phase before proposing a new one.

### State machine (LangGraph)
The per-page extraction loop and the per-section report agent are both
LangGraph `StateGraph`s — a mutable state object flows through nodes, and
a conditional router decides the next hop.

- Per-page: [app/agents/graph.py](../app/agents/graph.py) —
  `StateGraph(PageState)`; router `_after_extractor` returns
  `"pass"` (→ `END`) or `"escalate"` (→ escalation node).
- Per-section: [app/agents/reports/section_agent.py](../app/agents/reports/section_agent.py)
  — `plan → gather → (skip | reason) → draft → reflect → (revise | END)`.
- **Note:** this is *not* a Chain of Responsibility. Validation is folded
  into a self-validating extractor node
  ([app/agents/extractor.py](../app/agents/extractor.py)); escalation is a
  single conditional edge, not a handler chain.

### Blackboard
The agentic report run uses an explicit shared-working-memory blackboard.
The master is the **sole writer**; section agents only read scoped views.

- [app/agents/reports/blackboard.py](../app/agents/reports/blackboard.py)
  — writes (`set_agenda`, `add_digest`, `add_review_note`) vs. reads
  (`scope_for`, `view_for`). Serialized to `strategic_reports.blackboard`
  (JSONB).
- **Why:** keeps cross-section coordination auditable and replayable
  (see [15-report-generation.md](15-report-generation.md)).

### Producer–consumer (bounded streaming)
Rasterized pages are produced into a bounded deque and drained by
concurrent page workers, so we never hold every rendered PNG at once.

- [app/pipeline/orchestrator.py](../app/pipeline/orchestrator.py) —
  `PageDeque` (bounded) fed by the rasterizer, consumed by ~60 concurrent
  asyncio page workers.
- **Why:** memory is structural, not leaked — the bound is what keeps the
  render burst flat (see ingestion memory notes in
  [03-ingestion-pipeline.md](03-ingestion-pipeline.md)).

### Repository
Every database aggregate gets one module under `app/db/queries/` that opens
a session, runs the query, and returns plain dicts/rows. ORM types do not
leak past this layer.

- [app/db/queries/](../app/db/queries/) — `project_documents.py`,
  `document_meta.py`, `credits.py`, `usage.py`, `chat_sessions.py`,
  `feedback.py`, …
- **Why:** callers (agents, routes) never import SQLAlchemy models or hold
  sessions; ownership scoping (`owner_id`) lives in one place per aggregate.
- **Adding a query:** put it in the matching repository module; do not run
  `select(...)` inline in a route or agent.

---

## GoF patterns

### Singleton
Heavy, concurrency-capping resources are constructed once per process and
shared. The semaphore inside the shared client is what enforces the global
concurrency cap.

- [app/utils/gemini_client.py](../app/utils/gemini_client.py) — one
  `genai.Client` shared by all workers, capped by an
  `asyncio.Semaphore`.
- [app/db/engine.py](../app/db/engine.py) — `_engine` /
  `_session_factory` module globals, lazily created.
- [app/main.py](../app/main.py) — Gemini client, embedder, and Qdrant store
  wired once into `app.state` at lifespan startup.

### Adapter
Third-party SDKs and swappable backends sit behind a stable project-local
API, so the rest of the code never imports the vendor type directly.

- [app/storage/local_storage.py](../app/storage/local_storage.py) —
  `LocalStorage` is a drop-in replacement for the former Supabase storage:
  same public API, same path scheme.
- [app/vectorstore/qdrant_store.py](../app/vectorstore/qdrant_store.py) —
  `QdrantVectorStore` wraps `AsyncQdrantClient`.
- [app/vectorstore/embedder.py](../app/vectorstore/embedder.py) —
  `GeminiEmbedder` wraps the Gemini embedding API.
- **Why:** the Supabase→local migration only had to swap the adapter, not
  every call site.

### Factory / provider
Cached or lazily-built shared resources are handed out through provider
functions, never constructed at the call site.

- [app/config.py](../app/config.py) — `get_settings()` returns a cached
  `Settings`.
- [app/db/engine.py](../app/db/engine.py) — `get_engine()` /
  `get_session_factory()` lazy factories.
- [app/vectorstore/qdrant_store.py](../app/vectorstore/qdrant_store.py) —
  `collection_names()` resolves the `(chunks, entities, images)` triple for
  a tenant.

### Dependency injection
Collaborators are passed in (constructor / keyword args / deps bundles)
rather than reached for globally. This is pervasive and intentional.

- [app/agents/graph.py](../app/agents/graph.py) — `build_graph(client,
  settings)` captures injected deps in node closures.
- [app/agents/rag_tools.py](../app/agents/rag_tools.py) — tools take
  injected `embedder` and `vector_store`.
- [app/agents/reports/section_agent.py](../app/agents/reports/section_agent.py)
  — `SectionAgentDeps` bundles `client/settings/embedder/vector_store/
  user_id/…`.
- FastAPI request-time injection via `app.state`
  ([app/main.py](../app/main.py)).

### Strategy
Interchangeable algorithms selected at runtime — often with the LLM as the
selector.

- [app/agents/rag_tools.py](../app/agents/rag_tools.py) — `search_chunks`,
  `search_entities`, `search_images` share a signature shape; the model
  picks which to call.
- [app/retrieval/tier_weighting.py](../app/retrieval/tier_weighting.py) —
  `_TIER_WEIGHTS` maps artifact class → scoring weight via
  `get_tier_weight()`.
- [app/agents/graph.py](../app/agents/graph.py) — escalation model is
  chosen by page type (`is_scanned`).

### Registry
A keyed lookup with a built-in cache and a fallback source.

- [app/agents/reports/registry.py](../app/agents/reports/registry.py) —
  `BUILTIN_REGISTRY` of `ReportTemplateSpec`; `resolve_template()` does
  built-in → DB fallback.
- [app/retrieval/tier_weighting.py](../app/retrieval/tier_weighting.py) —
  `_TIER_WEIGHTS` keyed by artifact class.

### Facade
A single entry point hides multi-tool / multi-stage complexity from callers.

- [app/agents/rag/retriever.py](../app/agents/rag/retriever.py) —
  `rag_search` fans out across the three collection tools in parallel and
  returns one deduped, sorted list.
- [app/agents/rag/agent.py](../app/agents/rag/agent.py) — prefilter →
  guard → synthesizer behind one chat entry point.

### Template Method
A fixed skeleton calls a per-case primitive.

- [app/vectorstore/qdrant_store.py](../app/vectorstore/qdrant_store.py) —
  `ensure_collection()` calls `_ensure_single_collection()` three times
  with per-collection index maps.
- [app/vectorstore/entity_indexer.py](../app/vectorstore/entity_indexer.py)
  and `image_indexer.py` share a build → embed → upsert skeleton.

### Command (deferred dispatch)
A request is encapsulated as a deferrable Celery task object.

- [app/agents/rag/agent.py](../app/agents/rag/agent.py) —
  `extract_user_memory.delay(...)` dispatches work to the worker
  fire-and-forget.

### Value Object / DTO
Typed, often frozen, structured transfer objects with explicit mappers
between them.

- [app/pipeline/page_router.py](../app/pipeline/page_router.py) —
  `RouteDecision(frozen=True)` with `to_meta()`.
- [app/agents/rag_tools.py](../app/agents/rag_tools.py) — `SearchResult`,
  mapped from tier-weighted payload dicts via `_to_search_results()`.
- [app/vectorstore/entity_indexer.py](../app/vectorstore/entity_indexer.py)
  — `EntityPoint`.

---

## Patterns we deliberately do *not* use

- **Chain of Responsibility** for extraction — replaced by a self-validating
  extractor node + one conditional escalation edge (above).
- **Observer (GoF)** for progress — propagation is DB-row writes + SSE /
  polling, not subject/observer registration. The FE polls
  `GET /ingest/{job_id}` (see [01-architecture.md](01-architecture.md)).
- **New parallel abstractions** — `QdrantVectorStore.for_org()` is a
  flyweight-style *view* (shares the client, rebinds collection names), not
  a fresh instance. Prefer extending such views over new classes.

## When you're tempted to add a pattern

1. Check this page for an existing pattern that fits.
2. If one fits, reuse it and link the new code to the same module.
3. If nothing fits, the default answer is **no** — confirm with the owner
   before introducing a new abstraction
   ([.github/copilot-instructions.md](../.github/copilot-instructions.md)).
