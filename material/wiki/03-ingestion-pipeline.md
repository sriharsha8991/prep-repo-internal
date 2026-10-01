# 03 — Ingestion Pipeline

> _Changed 2026-07-30: refreshed for the current realtime lane — the vector-figure
> and rotated-page routing rules, the never-ship-a-blank-page guard, broken
> text-layer detection, per-page resume + cross-user dedup, the WITSML domain lane
> (Lane D), and Phases 6–8 (Well Knowledge Layer). Owns: source → page-level
> markdown / images / tables / entities / facts, and the Celery topology that
> drives it._

For the module-by-module walk-through see
[app/pipeline/README.md](../app/pipeline/README.md) (architecture) and
[app/pipeline/CLAUDE.md](../app/pipeline/CLAUDE.md) (operational rules + tuning
history). Historical narrative lives in
[INGESTION_FLOW.md](../data/plan_documentations/INGESTION_FLOW.md). This page owns
the *why* and the operational shape.

## Three lanes

Every upload picks a **lane** (`realtime` default, or `batch`). A third lane is
chosen by the pipeline itself from the file's contents.

| Lane | Chosen by | Model calls | Escalation | Turnaround |
|---|---|---|---|---|
| **realtime** | `lane="realtime"` + PDF bytes | synchronous online Gemini, full price | yes | minutes |
| **batch** | `lane="batch"` (**PDF only**) | Gemini Batch API, ~50% cheaper | **no** — tiers the model instead | hours (≤24 h) |
| **domain (Lane D)** | content sniff says WITSML/XML | **none** | n/a | milliseconds |

- Lane D is **not user-selectable**. `source_adapters.detect_source_kind()` sniffs
  bytes, never the extension — local storage names every source `source.pdf`, so by
  the time a worker sees the file the extension is a lie.
- `lane="batch"` + `.xml`/`.witsml` is rejected at the route with
  `error_code: lane_unsupported_for_format`: the batch lane exists to batch *vision*
  calls, and a WITSML DDR has no pages. `ingest_batch_submit` re-checks anyway and
  re-routes such a message to `ingest_document`.
- Accepted extensions come from `source_adapters.SUPPORTED_SUFFIXES`
  (`.pdf`, `.xml`, `.witsml`) so the upload gate and the lane router can never
  disagree. See [PLAN_v3_ingestion.md](../data/plan_documentations/PLAN_v3_ingestion.md).

## Realtime stages (live request path)

```
POST /ingest  or  /ingest/upload             app/routes/ingest.py
  ├─ authorize project (load_and_assert "ingest") → resolve the PROJECT's org
  ├─ validate artifact_class / confidentiality / lane / lane-vs-format
  ├─ spool upload → temp file, sha256 while streaming (constant memory)
  ├─ dedup: documents.content_hash + project_id → "duplicate" unless last job failed
  ├─ INSERT documents + jobs (status=queued, lane=…)
  ├─ reserve credits (ref_id = job_id)
  ├─ store source → pdfs bucket as `source.pdf`
  └─ dispatch Celery `ingest_document` (task_id=job_id) → queue "extract"
        └─ PipelineOrchestrator.run(...)      app/pipeline/orchestrator.py
             ├─ lane sniff → PDF path, or Lane D `parsing_domain_source`
             ├─ dedup fast path: all pages cached for this content_hash ⇒ skip 1+2
             ├─ Phase 0  preprocessing  — fitz baseline + metadata (~1 s)
             │            + background liteparse enrichment (layout-aware md)
             ├─ Phase 1  rasterizer     — stream PNGs (adaptive DPI) → PageDeque(32)
             ├─ Phase 2  worker pool    — 60 asyncio workers: resume? → route →
             │            LITEPARSE | CLASSIFY | VISION → guard → persist state
             ├─ Phase 3  postprocessor  — cross-page table merge, full_document.md
             ├─ Phase 4  vectorize      — chunk → embed → Qdrant (chunks/entities/images)
             ├─ Phase 5  auto-classify  — artifact_class if the user left "other"
             ├─ Phase 6  insight synth  — Layer A → document_insights (parallel parts)
             ├─ Phase 7  projection     — Layer A → npt_events/lessons/fact_edges (no LLM)
             ├─ Phase 7b ddr projection — typed DailyReports → daily_reports/activity_intervals
             └─ Phase 8  rollup         — top-5 NPT + Lessons → project_insights (no LLM)
  └─ Job row → succeeded | failed; usage_events row; credits settled or released
```

**Phases 0–4 are load-bearing. Phases 5–8 are best-effort** — each is wrapped so a
failure is logged and swallowed; the document is already usable and searchable.

Status writes happen at start/end; a **progress heartbeat** flushes
`current_phase` / `pages_total` / `pages_done` every
`progress_write_interval_seconds` (3 s). The FE **polls** `GET /ingest/{job_id}`
(no Supabase Realtime — that went with the local migration).

`current_phase` values the FE can see: `queued`, `starting`,
`parsing_domain_source`, `preprocessing`, `streaming_extraction`, `postprocessing`,
`vectorizing`, `insight_synthesis`, `insight_projection`, `ddr_projection`,
`insight_rollup`, `done` — plus the batch-only `batch_submitting`,
`batch_processing`, `batch_collecting`.

## Why per-page streaming and not batch (within a doc)

PDFs are I/O- and LLM-bound, not CPU-bound. The orchestrator starts 60 asyncio
workers *before* the rasterizer so each rendered page is extracted the moment it
lands. ~91 % of wall-clock is waiting on the Gemini API; rasterize / postprocess /
chunk are a small CPU fraction. Do **not** collapse this into a synchronous
per-document batch.

## Stage details

### Phase 0 — Baseline text · [preprocessing.py](../app/pipeline/preprocessing.py)
- `extract_fast` (fitz only, ~1 s): per-page raw text, `char_count`, `image_count`,
  `rotation`, width/height, `is_scanned` (text < 50 chars).
- `enrich_baselines` runs **in the background** (LiteParse — Rust+PDFium spatial
  text) upgrading each non-scanned page's `.text` to layout-aware markdown; awaited
  before Phase 3.
- **Broken text layers** (`text_layer_is_unusable`): a subset-embedded font with a
  missing ToUnicode CMap makes `get_text()` return glyph codes (`"\x02\x03\x04…"`).
  The page looks text-rich, takes the cheap route, and pushes control-character
  garbage into the markdown, chunker and embeddings — worse than blank, because it
  silently pollutes retrieval. Detected pages get a blanked baseline and are marked
  scanned so they go to vision. `char_count` keeps its original value so the
  blank-page rule doesn't skip them. Measured: 4 of 23 pages in one real WCR, zero
  false positives across 175 other text pages.
- Table detection is deliberately **not** here — see Phase 1.

### Phase 1 — Streaming rasterizer · [page_streamer.py](../app/pipeline/page_streamer.py)
- Renders pages to PNG in 8 bounded threads (`render_concurrency`), pushing each to
  a `PageDeque` (maxsize 32 ≈ 6 MB in flight) immediately. Ordered sliding window
  preserves FIFO page order without piling up PNGs behind one slow page.
- **Adaptive DPI**: scanned or dense small print → 300, sparse → 150, else 200.
- Runs `find_tables()` on liteparse-candidate pages, counting only **substantial**
  tables (≥ `route_table_min_rows` × `route_table_min_cols`, default 2×2) so a
  single bordered box doesn't force a paid VISION call.
- Counts **vector drawings** (`get_cdrawings()`) — the signal the router needs for
  CAD-style schematics, and what lets the empty-page guard tell a figure-bearing
  page apart from a blank sheet.
- Every PNG is persisted to the `pages` bucket as it is rendered.

### Phase 2 — Worker pool + page router · [orchestrator.py](../app/pipeline/orchestrator.py) `_worker`, [page_router.py](../app/pipeline/page_router.py)

Routing is one shared pure function (`decide_route`) used by both lanes. First
match wins:

| Rule | Route | Why |
|---|---|---|
| blank: no text, no image, no vector | LITEPARSE | separator sheets — nothing to extract, don't pay |
| `is_scanned` | VISION | no text layer |
| `image_count > 1` | VISION | figure-rich page |
| `drawing_count >= route_vector_drawing_min` (60) | VISION | vector figure |
| `rotation != 0` | VISION | sideways text defeats baseline line grouping |
| `table_count > 0` (substantial) | VISION | structured extraction needed |
| `char_count < route_text_rich_chars` (300) | CLASSIFY | flash-lite arbiter decides |
| else | LITEPARSE | trust the baseline, **0 LLM calls** |

- `image_count` counts **raster XObjects only**. A CAD-style well schematic or
  drilling-program summary is drawn with path operators, so it reports
  `image_count == 0` while carrying enough label text to clear
  `route_text_rich_chars` — those pages used to take LITEPARSE and ship scrambled
  labels or nothing. `route_vector_drawing_min` catches them; measured on real
  reports it moves ~2 pages per 23-page document.
- **Liteparse promotion**: a LITEPARSE verdict whose baseline is too thin is
  promoted to VISION (`worker_liteparse_promoted_to_vision`). Accepting it would
  ship a blank page as a success; the cost is one page.
- **Resume**: a clean per-page `page_NNN.state.json` from a prior run is reused
  (skips the LLM calls). State is written as each page completes, so a job that died
  at page 1499/1500 doesn't re-extract the document. Degraded/error pages get a
  fresh attempt. Liteparse pages persist state too, so a resume doesn't redo them.
- **Cross-user dedup**: page extractions are cached in `page_extractions` by
  `(content_hash, page_num)` when `ingest_dedup_enabled`. If every page is cached,
  Phases 1+2 are skipped entirely — no rasterizing, no Gemini calls.
- Failures fall back to baseline text and are flagged `degraded` (surfaced when
  `degraded_ratio ≥ ingest_max_degraded_ratio`).
- PNG bytes are dropped right after the model call; only a trimmed resident copy of
  each page state stays in RAM (the verbose trace lives in the state JSON).

### Extractor → (escalation) · [graph.py](../app/agents/graph.py), [extractor.py](../app/agents/extractor.py), [escalation.py](../app/agents/escalation.py)
- **There is no separate `validator.py`** — the extractor **self-validates** in one
  call (returns `self_confidence` + `self_gaps` alongside the extraction).
- **Extractor** = `extraction_model` (flash-lite): one multimodal call → markdown,
  tables, images, entities, cross_page_notes + self-assessment. Entity normalization
  (controlled vocab) and numeric guardrails append gaps and can force escalation.
- **Conditional edge** (`_after_extractor`) escalates when **any** of: the
  extraction is underfilled (checked *before* confidence — see below),
  `validation.passed` is false, or confidence < per-page threshold
  (digital 0.78 / scanned 0.85).
- **Escalation** = `escalation_model` (`escalation_model_scanned` for scanned
  pages), re-extracting with the prior JSON + gaps as targeted feedback. FINAL.
- All Gemini calls go through [GeminiClient](../app/utils/gemini_client.py): shared
  `Semaphore(max_concurrent_pages)`, adaptive pacing/backoff on 429, 120 s timeout,
  3 retries.

### Never ship a blank page · [extraction_guard.py](../app/pipeline/extraction_guard.py)
Every route could previously emit an empty page and nothing checked: the graph's
exit edge read only the model's *self-reported* confidence, so a
confident-but-empty answer (`{"markdown": "", "self_confidence": 0.95}` — what dense
figure pages provoke) went straight to END and rendered blank in the viewer while
the job reported success. Three nets now:

1. `graph._after_extractor` escalates an underfilled extraction regardless of
   self-reported confidence.
2. The worker refuses a LITEPARSE verdict whose baseline is too thin and promotes
   the page to vision.
3. After the graph, a still-empty page keeps whatever baseline text exists and is
   flagged `degraded` + `empty_extraction` — counted in the job summary
   (`empty_pages`, `empty_page_numbers`) and logged as `pipeline_empty_pages`.

`is_underfilled` has three verdicts: `empty_extraction` (nothing came back),
`thin_vs_baseline` (effectively nothing, on a page the baseline proved had text),
and `dropped_vs_baseline` (< `extraction_min_baseline_ratio` = 25 % of the
baseline — proportional, because 500 chars against a 4000-char baseline sails past
any fixed floor). `looks_blank()` (no text, no image, no vector) is what stops
separator sheets from burning an escalation on every one of them.

### Phase 3 — Postprocessor · [postprocessor.py](../app/pipeline/postprocessor.py)
- Cross-page table merge (`continues_on_next_page`, adjacent pages + matching
  column count), per-page `page_NNN.md`, `full_document.md` with `<!-- Page N -->`
  markers (Phase 6 splits on those), per-page and merged table JSON/markdown.
- Header/footer detection is tiered: skipped under 10 pages, 70 % co-occurrence at
  10–29, 50 % at 30+.
- Entity resolution → `entities.json` (raw + resolved + canonical);
  `resolved_by_page` is kept in memory for Phases 4b and 6.

### Phase 4 — Vectorize · [app/vectorstore/](../app/vectorstore/)
- `delete_by_document` runs **first**, so re-ingest is idempotent even when the new
  parse yields zero chunks.
- `chunker` → ~1000-char chunks (overlap 200); tables are never sliced. `embedder` =
  `gemini-embedding-001` (768-d), embedded and upserted in **bounded windows** so a
  large document never holds every vector in RAM.
- 4b entity points, 4c image-description points. All I/O scoped to the tenant's
  org collections (`for_org_id`, falling back to `for_org` for in-flight legacy
  messages).

See [05-storage-and-vectors.md](05-storage-and-vectors.md) for collection payloads.

### Phase 5 — Auto-classify · [artifact_classifier.py](../app/pipeline/artifact_classifier.py)
Only when `artifact_class` is still the default `"other"`. Realtime threads the
uploaded class through (no extra `SELECT`) and reuses the just-assembled markdown
(no storage re-read); the `UPDATE` is still guarded on `artifact_class = 'other'` so
a concurrent manual edit can't be clobbered. An unsure classifier keeps the default.

### Phases 6–8 — Well Knowledge Layer
See [17-well-knowledge-layer.md](17-well-knowledge-layer.md) for the data model.
Ingestion-side shape:

- **Phase 6 (Layer A)** — [insight_extraction.py](../app/pipeline/insight_extraction.py):
  ontology-driven NPT / lessons / wells / edges extraction over data already in hand
  (`full_md` + resolved entities); **no Qdrant calls**. It is
  **divide-and-conquer over pages**: parts of `insight_extraction_pages_per_part`
  (5) pages extracted **in parallel** (max 6 concurrent, 60 parts), each carrying
  only its own pages' entity anchors, then merged and deduped. Measured, not
  stylistic: on a 32-page report 4 pages/part yielded 17 NPT + 5 lessons vs
  19 pages/part yielding 11 + 1. Do not collapse it into one call because the
  context window fits — the window fits, recall drops. Each part retries once; a
  result with `part_failures > 0` is a *partial* extraction and logs
  `insight_synthesis_partial`. Usage bills to the user via `job.extra_usage`.
- **Phase 7 (Layer B)** — [insight_projector.py](../app/pipeline/insight_projector.py):
  pure re-projection, no LLM. Wipe-and-replace per document into `npt_events`,
  `lessons`, `fact_edges` with provenance (`document_id`, `page_num`, `confidence`).
- **Phase 7b (DDR)** — [ddr_projector.py](../app/pipeline/ddr_projector.py): the
  domain lane's typed records → `daily_reports` + `activity_intervals`. Same-day
  collisions (day + night tour sheets) are summed before insert. Runs **outside**
  the `insight_extraction_enabled` gate — these facts come from a deterministic
  parse, so turning off LLM insight extraction must not discard exactly-parsed
  depth-per-day data.
- **Phase 8 (rollup)** — [insight_rollup.py](../app/pipeline/insight_rollup.py):
  deterministic SQL reduce to top-5 NPT + top-5 lessons in `project_insights`,
  skipped when `source_hash` is unchanged; superseded documents are excluded
  everywhere (including from the hash, so the cache self-heals).
  **Note:** the Phase 8 block currently sits inside the `if job.daily_reports:`
  branch, so at ingest time it only runs for domain-source documents. PDF ingests
  are rolled up lazily — `GET /projects/{id}/insights` and the report facts block
  call `rollup_project()` on read.

Well/formation identity is gated off (`well_registry_enabled = False`): string
similarity mis-merges numeric-suffix pad names (Sajaa 6 vs Sajaa 9 at 0.857). Facts
still populate with NULL `well_id`/`formation_id`, so the dashboards work honestly.

## Lane D — WITSML / XML daily drilling reports · [source_adapters.py](../app/pipeline/source_adapters.py)

The pipeline is PDF-shaped only up to Phase 3. Everything from the postprocessor
rightward needs just **two structures**: `list[PageBaseline]` and
`page_results[page] = {"final": {markdown, tables, images, entities}}`. That is the
seam an adapter targets — hit it and you get chat, search, NPT/lessons extraction,
reports and RCA for free without touching a downstream phase.

Lane D therefore has **zero rasterization, zero vision tokens, and deterministic
output**: the numbers are read from typed WITSML elements, so the same file always
produces the same facts. One synthetic page is emitted per reporting day, which
keeps `page_num` citations meaningful ("day 3" reads as page 3). Phases 0–2 and the
page cache are skipped; Phase 7b projects the typed records into Layer B, which is
what the time-depth benchmark reduces over.

Adding a new format means **adding an adapter here**, not a new pipeline.

## Batch lane · [batch_orchestrator.py](../app/pipeline/batch_orchestrator.py)

```
ingest_batch_submit (queue "rasterize")     batch_submitting → batch_processing
  Phase 0 + Phase 1, then route each page AS IT LANDS:
    LITEPARSE pages skip the batch (baseline text filled in at collect)
    others → one request each, grouped by tier (digital | scanned),
             flushed as an inline sub-job at batch_inline_max_bytes (18 MB)
  bookkeeping → jobs.batch_meta; task EXITS (never blocks a worker for ~24 h)

poll_batch_job (queue "batch_poll", every batch_poll_interval_seconds = 300 s)
  refresh each sub-job → running | succeeded | failed; self-reschedules;
  gives up after batch_max_poll_hours (30 h) → failed / batch_state "expired"

collect_batch_job (queue "batch_collect")   batch_collecting → shared Phases 3-7
  re-read PDF + full liteparse, fetch inline responses, parse into the realtime
  per-page shape, persist state + cache, then orchestrator.finalize()
```

The batch lane has **no escalation loop**, so quality rests on a single strong-model
pass; scanned/image-heavy pages get `batch_extraction_model_scanned`. Cost reporting
applies `batch_cost_discount` (0.5) to generation only — embeddings are always
online. Both poll and collect are idempotent: a terminal job short-circuits, and a
transient failure re-schedules rather than stranding the job.

## Where the data lands

Filesystem (`storage_root`, tenant-partitioned —
`<bucket>/<user_id>/<project_id>/<document_id>/`):

| Bucket | Files |
|---|---|
| `pdfs` | `source.pdf` (always this name, whatever was uploaded) |
| `pages` | `page_NNN.png` |
| `artifacts` | `page_NNN.md`, `page_NNN.state.json`, `full_document.md`, `entities.json`, `tables/page_NNN_table_MM.{json,md}` |

Postgres: `documents`, `jobs` (+`batch_meta`), `page_extractions` (shared cache),
`usage_events`, credit holds/ledger, `document_insights` (Layer A JSONB),
`npt_events` / `lessons` / `fact_edges`, `daily_reports` / `activity_intervals`,
`project_insights`. Qdrant: org-scoped chunks / entities / images collections.
Details in [05-storage-and-vectors.md](05-storage-and-vectors.md) and
[06-database-schema.md](06-database-schema.md).

**Ordering gotcha:** `page_NNN.state.json` appears as each page completes;
`full_document.md` only exists after Phase 3 has processed every page.

## Celery topology · [celery_app.py](../app/workers/celery_app.py)
- Broker **and** result backend = Redis (`/0` broker, `/1` results, `/2` app state).
- One document = one `ingest_document` task = one prefork child. Intra-doc
  parallelism is asyncio (60); inter-doc parallelism is `--concurrency=4`.
- Queues: `extract`, `rasterize`, `aggregate`, `reports`, `default` on
  `celery-worker`; `batch_poll`, `batch_collect` on the dedicated
  `celery-batch-worker`. `acks_late=True`, `--max-tasks-per-child=5`,
  `--prefetch-multiplier=1`.
- Each task runs the async pipeline in a fresh event loop and disposes the
  SQLAlchemy async engine in `finally:`.
- Credits: reserve at the route by `ref_id = job_id`; the task settles with the
  **measured** tokens or releases on failure — exactly once per exit path.

## Known scaling limits (see plans)
- One huge doc is pinned to one child → single-worker bottleneck + memory growth.
  Mitigation plan: [PLAN_pdf_sharding.md](../data/plan_documentations/PLAN_pdf_sharding.md).
- A bulk upload of N documents recomputes the project rollup N times (per-doc,
  synchronous). Fine at current volume; debounce via a queued dirty flag once a
  single upload routinely exceeds ~10 documents.

## Concurrency notes that bite
- `render_concurrency = 8` is a long-stable value, not a placeholder. PyMuPDF
  rasterization (incl. `find_tables`, which holds the GIL) is CPU-bound; raising it
  oversubscribes CPU and **starves the asyncio loop servicing in-flight Gemini
  sockets**, producing clustered call failures that look like API problems.
- `max_concurrent_pages = 60` per process; cluster ceiling is that × Celery
  concurrency. 100 saturated the API tier and produced 120 s timeouts.
- Gemini call *starts* are spaced by `gemini_inter_call_delay_seconds`, widened
  adaptively on 429 and decayed back, so the whole page fleet backs off rather than
  just the rate-limited call.

## Observability
Structured (structlog) events worth grepping:
```
pipeline_preprocessing_fast_done, baseline_text_layer_unusable, page_rasterized,
worker_page_resumed_state, worker_page_classified_liteparse,
worker_liteparse_promoted_to_vision, extractor_done, extractor_underfilled,
escalation_done, gemini_call_ok, gemini_call_error,
worker_page_complete, worker_page_failed, worker_page_empty_after_escalation,
pipeline_dedup_cache_hit, pipeline_extraction_done, pipeline_postprocessing_done,
pipeline_vectorize_done, pipeline_degraded_pages_high, pipeline_empty_pages,
insight_extraction_done, insight_synthesis_partial, ddr_projector_done,
rollup_project_done, pipeline_complete, pipeline_failed,
batch_submit_done, batch_collect_done
```
Spans: `ingest.reserve_credits`, `ingest.store_pdf`, `ingest.dispatch_task`,
`ingest.domain_parse`, `ingest.preprocess.fast`, `ingest.preprocess.enrich`,
`ingest.rasterize`, `ingest.finalize`, `ingest.postprocess`, `ingest.vectorize`,
`ingest.insight_synthesis`, `ingest.insight_projection`, `ingest.ddr_projection`,
`ingest.insight_rollup`.

Never log prompts, chunks, document text, or model answers.

## Things this page does not own
- Where bytes physically go → [05-storage-and-vectors.md](05-storage-and-vectors.md).
- `jobs` / `documents` columns → [06-database-schema.md](06-database-schema.md).
- Serving artifacts to the viewer → [13-document-viewer-and-serving.md](13-document-viewer-and-serving.md).
- How chat consumes artifacts → [04-retrieval-and-chat.md](04-retrieval-and-chat.md).
- NPT / lessons / RCA / time-depth data model → [17-well-knowledge-layer.md](17-well-knowledge-layer.md).
