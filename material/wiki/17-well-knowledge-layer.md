# Well Knowledge Layer — NPT + Lessons Insights

> _Added 2026-07-14._ Owns: ingestion-time extraction of **NPT events** and
> **Lessons Learned** into structured tables, the per-project **top-5 rollups**,
> and the read/RCA APIs that serve them. For the raw PDF→markdown/entities
> pipeline see [Ingestion Pipeline](03-ingestion-pipeline.md); for the tables'
> DDL see [Database Schema](06-database-schema.md).

The knowledge layer turns every ingested well report into two things the
product sells: **what cost us time (NPT)** and **what we learned (Lessons)**.
It is a shift from *query-time* extraction (ask Qdrant + LLM every time) to
*ingestion-time* enrichment (extract once, read many). Downstream — dashboard,
chat, reports, RCA — become cheap reads.

## Scope (2026-07): NPT + Lessons only

Wells and formations are extracted and the tables exist, but **well-identity
resolution is gated OFF** (`settings.well_registry_enabled = False`). String
similarity mis-merges well-pad families (e.g. `Sajaa 6` vs `Sajaa 9` at 0.857),
so we ship NPT + Lessons — which aggregate correctly by category/project/
document and need no well identity — and defer wells to a domain-aware resolver.
Everything below therefore populates `npt_events` + `lessons`; `wells`,
`formations`, `fact_edges`, `well_insights` stay empty until the flag flips.
See [WELL_KNOWLEDGE_LAYER_PLAN.md](../frontend_docs/WELL_KNOWLEDGE_LAYER_PLAN.md)
(phase **P-Well**).

## NPT categories are a CLOSED vocabulary

Unlike the rest of the ontology (open vocab, `other` escape hatch), the NPT
category axis is **closed** — it's a standard (IADC), and letting the model
invent values did real damage. Live data before the fix:

```
BHA failure          11 events  104.9h    <- same category,
bha_failure           2 events   93.1h    <- split into two rows

Mechanical failure  /  mechanical_failure  /  Equipment failure  /  equipment failure
   ^ four variants of one thing; plus junk: "Baker", "SLB", "Drilling", "Trouble time"
```

The top-5 ranking was meaningless. Now ([npt.py](../app/domain/npt.py)):

- **`category`** is always one of the closed set → safe to `GROUP BY`. Enforced
  twice: the extraction prompt injects the explicit list ("copy one verbatim;
  a category is a FAILURE TYPE, never a company/vendor/tool brand"), and the
  **projector canonicalizes anyway** (`canonicalize()` — normalized exact →
  synonym → longest-substring synonym → `other`), so a sloppy model can't
  corrupt the fact table.
- **`category_raw`** keeps the report's own wording whenever it differed
  (`Short in BHA #6…`, `Problem with injector brake`, `Orienter failure`).
  **Closing the vocabulary drops nothing** — unmappable values land in `other`
  with their wording intact, and `category_raw` is the mining ground for
  categories worth promoting into the set later.

The set is a strict superset of the legacy `well_taxonomy.NPT_CATEGORIES`, so
every legacy value canonicalizes to itself and the per-page extractor path is
untouched. After the fix, the same corpus collapsed from **18 category strings
to 8 canonical ones**, and `bha_failure` correctly totals 238h across 3 docs.

## The three layers

```
  ONTOLOGY                app/domain/ontology.py
    3 focus nodes (WELL / NPT / LESSON) + `other`, typed edges, 1-/2-hop
    capture radius. Open vocabulary — values outside the lists are KEPT,
    never dropped. Drives the extraction prompt.
        │
        ▼
  LAYER A — raw            document_insights  (one JSONB row per document)
    The extractor's verbatim JSON: npt_events[], lessons[], wells_mentioned[],
    edges[]. Wells/formations by NAME. Immutable provenance; re-projectable
    without re-running the LLM. schema_version = ontology version.
        │  PROJECTOR (no LLM)
        ▼
  LAYER B — normalized     npt_events, lessons  (+ fact_edges, gated)
    One typed row per fact, FK to project + document (+ well/formation when the
    registry is on). Carries hours, category, cause, confidence, page_num.
    Wipe-and-replace per document → idempotent.
        │  ROLLUP (deterministic; lessons dedup)
        ▼
  ROLLUP — cached          project_insights  (one row per project)
    top_npt (top-5 by summed hours), top_lessons (top-5 by confidence).
    source_hash → skip recompute when the contributing facts are unchanged.
```

## Where it runs in ingestion

Three new phases at the **end** of `PipelineOrchestrator.finalize()`
([orchestrator.py](../app/pipeline/orchestrator.py)), after the document is
already searchable — so they never delay chat/search and never fail an ingest
(all best-effort). Gated by `settings.insight_extraction_enabled`.

| Phase | Step | LLM? | Reads | Writes |
|---|---|---|---|---|
| **6** | Insight synthesis | ✅ 1+ parallel calls | `full_md` + `resolved_by_page` | `document_insights` (Layer A) |
| **7** | Projection | ❌ | `document_insights` | `npt_events`, `lessons` (Layer B) |
| **8** | Rollup | ❌ | `npt_events`, `lessons` | `project_insights` (cached) |

**Failure never destroys good data.** A `failed` extraction result cannot
overwrite a previously successful `document_insights` row (conditional upsert
in [insights.py](../app/db/queries/insights.py)), and the projector skips its
wipe-and-replace when the insight state is `failed` — so a transient LLM outage
during a re-ingest leaves both layers at their last good state instead of
zeroing the document's facts.

### Phase 6 is divide-and-conquer, not one big call

The old design fed a **truncated** slice of the document markdown to one large
model — lossy on big PDFs (the tail is dropped) and prone to "lost in the
middle" recall loss. It's now a **map-reduce over document parts**
([insight_extraction.py](../app/pipeline/insight_extraction.py)):

1. **Split** `full_md` on its `<!-- Page N -->` markers, packing consecutive
   pages into parts of `insight_extraction_pages_per_part` pages (default **5**).
   The unit is pages, not chars — deterministic across page density. A part
   closes early if those pages would exceed the `insight_extraction_max_chars`
   char safety cap; a single monster page is clipped to it. Each part carries
   **only its own pages' entity anchors**, so `page_num` citations survive.

   > Why 5 and not "the whole doc"? flash-lite's 1M-token window *fits* the
   > whole document, but recall drops on large inputs ("lost in the middle").
   > Measured on a 32-page report: ~19 pages/part → 11 NPT + 1 lesson; **5
   > pages/part → 15 + 4** (7 calls); ~4 pages/part → 17 + 5 (11 calls). 5
   > pages cuts request count without entering the recall-degrading zone.
2. **Map** — every part is extracted **in parallel** (bounded by
   `insight_extraction_max_parallel`) on a small cheap model
   (`gemini-2.5-flash-lite`). A part that fails to parse is skipped, not fatal.
3. **Reduce** — fragments are concatenated and deduped (wells by name w/ alias
   union; NPT by category+page+cause; lessons by normalized text; edges by full
   tuple), then shape-cleaned into the Layer A row.

Small documents (≤ one chunk) take a single call — no chunking overhead. Large
documents simply spawn more flash-lite calls; the number of parallel calls is
capped by `insight_extraction_max_parts`.

**Cost:** each part is one `gemini-2.5-flash-lite` call (15× cheaper than flash
on input, 22× on output — $0.10/$0.40 vs $1.50/$9.00 per 1M); the token usage of
**all** parts is summed into the job's `extra_usage`
so it bills once through the existing ingest settle path (billed to the user's
credits). Even a 40-part document is a few cents. Phases 7 + 8 are pure SQL.

## Sequence diagram — extraction → serve

```
 Orchestrator        flash-lite ×N       Postgres                     Consumers
 (finalize)          (parallel parts)    (Layer A/B/rollup)           (dashboard/chat/reports/rca)
 ─────────────────────────────────────────────────────────────────────────────────────────────

  Phase 3–5 done
  have: full_md,
  resolved_by_page
      │
  ── PHASE 6: insight synthesis (divide-and-conquer) ─────────────────────────
      │  SPLIT full_md on <!-- Page N --> → parts of ~5 pages each
      │        (char safety cap), each with its own pages' entity anchors
      │        ┌───────────────────────────────────────────────┐
      │  MAP   │ part 1 ─ generate_text(JSON) ─►│ flash-lite    │  (all parts in
      │  (∥)   │ part 2 ─ generate_text(JSON) ─►│  temp 0.1     │   parallel, bounded
      │        │  …                             │  JSON mime    │   by max_parallel)
      │        │ part N ─ generate_text(JSON) ─►│               │
      │        └───────────────────────────────────────────────┘
      │  IN  (per part): ontology prompt + that part's anchors + its page slice
      │  OUT (per part): {wells[], npt_events[], lessons[], edges[]}
      │  REDUCE: merge + dedup fragments  →  _clean_records()
      │          (bad part skipped; usage of every call summed → job billing)
      │  state = processed | empty | failed
      │────────── upsert document_insights (JSONB) ───────►│
      │                                                    │  [Layer A row]
      │
  ── PHASE 7: projection (no LLM) ─────────────────────────────────────────────
      │────────── read document_insights ────────────────►│
      │◄───────── npt_events[], lessons[] (raw) ───────────│
      │  well/formation resolution: SKIPPED (registry gated off → FK NULL)
      │  wipe existing rows for this document, then insert:
      │────────── DELETE+INSERT npt_events, lessons ──────►│
      │                                                    │  [Layer B rows]
      │
  ── PHASE 8: rollup (no LLM) ─────────────────────────────────────────────────
      │────────── SELECT … GROUP BY category ─────────────►│
      │◄───────── top-5 NPT by SUM(hours), top-5 lessons ──│
      │  source_hash unchanged? → skip. else:
      │────────── upsert project_insights (cached) ───────►│
      │                                                    │  [rollup row]
      │
  ════════════════════════ later, on demand ══════════════════════════════════
                                                           │
   GET /projects/{id}/insights ───────── read project_insights ───────────────► Dashboard
   chat: fetch_project_insights tool ─── read npt_events/lessons ─────────────► Chat answer
   report generation (facts_block) ───── read rollup, seed evidence ──────────► Report section
   POST /projects/{id}/rca ───── SELECT evidence ─┬─ grounded research (allowlisted) ─┐
                                                  └─ 1 LLM reasoning pass ────────────┴─► RCA narrative
```

### Inputs / outputs at a glance

| Stage | Input | Output |
|---|---|---|
| Extraction (P6) | assembled `full_md`, per-page resolved entities, ontology hint | `InsightResult` → `document_insights` JSONB |
| Projection (P7) | `document_insights` row | typed `npt_events` + `lessons` rows |
| Rollup (P8) | Layer B rows for a project | `project_insights.top_npt` / `top_lessons` |
| Read API | `project_id` / `document_id` + Bearer token | cached top-5 / raw per-doc facts |
| RCA | `project_id` + optional `symptom` | structured evidence + markdown analysis w/ citations |
| RCA research (1.5) | `symptom` + NPT category names **only** (privacy boundary) | `[external]` note + allowlisted `sources[]` cited inline as `[S<n>]` |

## The top-5 rollup — why deterministic

`insight_rollup.rollup_project()` ([insight_rollup.py](../app/pipeline/insight_rollup.py)):

- **Top-5 NPT = `ORDER BY SUM(hours) DESC` (frequency tie-break)** — a pure SQL
  reduce. Reproducible and free; no LLM decides "top." Each row carries a
  `sample` (the highest-impact single event) for a preview + citation.
- **Superseded documents are excluded.** Every aggregation (and the global
  `/projects/insights` queries in [insights.py](../app/routes/insights.py)) joins
  `documents` and drops rows whose document has `superseded_by` set — the same
  governance rule retrieval applies in `rag_tools._apply_governance`. Without it,
  uploading a corrected revision (v2) beside the original (v1) double-counts every
  NPT hour and lesson. The exclusion is applied to the `source_hash` too, so
  superseding a document changes the hash and the cached rollup self-heals on the
  next read. The join adds no round trips — the query budget below is unchanged.
- **Top-5 Lessons = confidence-ranked with near-duplicate text suppression.**
  LLM semantic clustering is deferred until the corpus is large enough that
  near-dups pollute the top 5.
- **`source_hash`** = SHA-256 over per-document `(id, fact counts, SUM(hours),
  MAX(created_at))`. The timestamps are load-bearing: projection is
  wipe-and-replace, so **any** re-projection mints new `created_at` values and
  bumps the hash even when counts coincide; deleted documents drop out of the
  GROUP BY and bump it too. Unchanged hash → skip recompute.
- **The dashboard read self-heals.** `GET /projects/{id}/insights` always goes
  through `rollup_project` (cheap hash check → cached row in the common case),
  so document deletions, re-projections, and failed background rollups repair
  themselves on the next page load instead of serving stale citations.

### Query budget (measured)

Because the dashboard read now runs the rollup on every page load, its query
count is load-bearing. Keep it this way:

| Path | Queries | Notes |
|---|---|---|
| Cached read (the common case) | **2** | source-hash (one `UNION ALL` over both fact tables) + cached `project_insights` row |
| Full recompute | **7** | hash, existing-row, GROUP BY category, `DISTINCT ON` sample, lessons, org lookup, upsert |
| Chat `fetch_project_insights` | **2** | top NPT + top lessons |

Two things this deliberately avoids: the per-category **`sample` N+1** (five
extra queries — now one `DISTINCT ON`), and re-reading the whole org registry
per well name in the projector (pass a once-loaded `known` list to
`resolve_well` / `resolve_formation`). Composite indexes in migration `024`
back every `project_id + …` filter.

> Bulk-upload note: today the rollup runs synchronously per document, so a
> 40-doc upload recomputes the project 40× (cheap SQL, but O(N)). If a single
> upload routinely exceeds ~10 docs, debounce via a Celery-queued dirty flag —
> tracked in the plan doc.

## APIs

Mounted from [insights.py](../app/routes/insights.py); ACL via
`accessible_project_ids` + `load_and_assert` (same as `/search`). The three
GETs settle ~0 credits; RCA is metered. Full request/response contracts for the
FE live in [communication_from_backend.md §0.21](../frontend_docs/communication_from_backend.md).

| Method + path | Purpose | Notes |
|---|---|---|
| `GET /projects/{project_id}/insights` | One project's cached top-5 NPT + top-5 Lessons | Recomputes on cache-miss so a fresh ingest is never blank. `state`: `processed`/`empty`/`pending`. |
| `GET /projects/insights?breakdown=by_project` | Global rank across all accessible projects | True global aggregate, **not** a stitch of per-project top-5s. Global NPT rows carry `document_count` (no `sample`); `by_project[]` rows use the full shape. |
| `GET /documents/{document_id}/insights` | Raw Layer A NPT + Lessons for one document | Wells/formations by name. Never 404s on state. |
| `POST /documents/{document_id}/insights/reextract` | Re-run extraction → projection → rollup for one doc | Reuses the stored `full_document.md`; **no PDF re-ingest**. Makes prompt/model tuning cheap (one LLM pass, not rasterize+vision+embed). Runs without per-page anchors (not persisted). Requires **`ingest`** access (editor/owner). Metered: reserves the `insights` estimate, settles on measured tokens, writes a `usage_events` row — as does `/rca`. |
| `POST /projects/{project_id}/rca` | Root-cause analysis over NPT + Lessons, layered against published standards | Body `{ "symptom": "…" }` (optional). Returns `evidence` (deterministic) + `analysis` (markdown, cited, with inline links to API/SPE/IADC/ISO/PetroWiki) + `sources[]`. See [RCA internet research](#rca-internet-research--strict-sites-inline-citations). |

**Citations use filenames, not UUIDs.** Every NPT/lesson row carries
`document_name` alongside `document_id`, so a citation reads
`(Sajaa 6 EOWR.pdf p.19)` — the RCA prompt, report facts block, and dashboard
all cite this way.

### Top-5 response shape (per-project)

```json
{
  "project_id": "…", "scope": "project", "state": "processed",
  "computed_at": "2026-07-13T17:02:25Z", "contributing_documents": 2,
  "wells": [],
  "top_npt": [
    { "category": "bha_failure", "total_hours": 35.15, "event_count": 3,
      "events_with_hours": 3, "avg_confidence": 0.95,
      "sample": { "cause": "BHA failure", "hours": 15.08, "date": "2004-10-16",
                  "document_id": "27dd…", "page_num": 19 } }
  ],
  "top_lessons": [
    { "lesson_id": "…", "text": "Set pH of base drilling fluid between 10.5 and 11.0…",
      "category": "fluids", "confidence": 0.9, "document_id": "27dd…", "page_num": 30 }
  ]
}
```

## Other consumers (no API of their own)

- **Chat** — `fetch_project_insights` tool in
  [synthesizer.py](../app/agents/rag/synthesizer.py) reads `npt_events` /
  `lessons` (via [insight_facts.py](../app/db/queries/insight_facts.py)) so
  NPT/Lessons questions answer from structured facts, not vector search.
- **Reports** — [facts_block.py](../app/agents/reports/facts_block.py) renders
  the rollup into the section analyst's `metrics_block`, so reports cite the
  same facts and spend fewer retrieval calls.
- **RCA** — [rca.py](../app/pipeline/rca.py): deterministic evidence assembly
  (cross-document recurrence via `npt_category_document_spread`) + one LLM
  reasoning pass over ~dozens of rows (not thousands of chunks), layered against
  published standards by the research pass below.

## RCA internet research — strict sites, inline citations

Operator evidence says *what happened*; it cannot say *which published mechanism
explains it* or *which limit was crossed*. Stage 1.5 of RCA
([rca_research.py](../app/pipeline/rca_research.py)) adds that layer: one
Google-Search-grounded call, restricted to authoritative sources.

**The allowlist** (`RCA_ALLOWED_DOMAINS`, ranked as a drilling engineer would
weigh them). Subdomains of an entry are accepted, so `spe.org` covers
`petrowiki.spe.org` and `jpt.spe.org`:

| Priority | Source | Purpose | Domains |
|---|---|---|---|
| ★★★★★ | API RP & Specs | Engineering rules and operating limits | `api.org`, `publications.api.org` |
| ★★★★★ | SPE OnePetro | Failure mechanisms, optimisation methods | `onepetro.org`, `spe.org` |
| ★★★★★ | IADC manuals (bit & BHA dull grading) | Drilling-specific operational guidance | `iadc.org`, `iadclexicon.org` |
| ★★★★ | ISO 16530 | Well lifecycle and integrity management | `iso.org` |
| ★★★★ | PetroWiki | Structured engineering knowledge | `petrowiki.spe.org` |

Three properties are load-bearing, and each exists because the obvious
implementation would silently violate it:

- **Strict sites are enforced, not requested.** Gemini's `GoogleSearch` tool has
  no site allowlist — only `exclude_domains` — so the `site:` terms in the
  prompt are *steering*. Enforcement is `_filter_sources`, which drops any
  chunk whose domain is off-list. Watch `rca_research_filtered` in the logs for
  kept/dropped domains — that log is how you tell a working filter from a
  vacuously satisfied one.
- **The prompt MUST mandate literal `site:` operators**, and this is the
  difference between the feature working and silently doing nothing. Measured
  against the live API: when the model paraphrases (`lost circulation causes
  PetroWiki`), Google returns wikipedia/scribd/researchgate/vendor blogs and the
  filter drops **every** source — tokens spent, nothing cited. When the model
  emits `site:onepetro.org …`, results come back **exclusively** allowlisted.
  Naming the sources in prose is not enough; `test_research_system_prompt_
  mandates_literal_site_operators` pins this.
- **Read the domain from `web.title`, not just `web.domain`.** `web.uri` is a
  `vertexaisearch` redirect and can never be parsed for the real host. Of the
  two remaining fields, the **Gemini Developer API leaves `web.domain` as
  `None`** and puts the bare domain in `web.title` (`"onepetro.org"`) — verified
  against gemini-3.1-flash-lite / google-genai 1.72. Vertex AI populates
  `domain` instead. `_chunk_domain` handles both, and fails closed: an unparsed
  domain is simply not on the allowlist.
- **Confidential data never reaches Google.** The grounded call is given only
  the symptom text and NPT category names — never well names, hours, costs,
  dates, or document titles, because its queries are derived from its prompt.
  This is why research is a *separate pass* rather than a tool on the reasoning
  call, and the boundary is commented as such in `run_standards_research`.
- **Citations cannot be hallucinated.** The model cites `[S<n>]` against a
  registry we built; `linkify` rewrites known markers into markdown links and
  strips unknown ones. The model never types a URL. Project claims keep the
  `(document_id p<page>)` scheme — the two are deliberately distinct.

External sources may explain mechanisms and supply limits; they must never
supply a fact about the operator's wells. **Thin project evidence stays thin** —
literature does not fill the gap. If no source survives the filter, the note is
discarded (prose with no citable source is ungrounded by construction) and RCA
proceeds on project evidence alone.

`analysis` carries the inline links; the response also returns a structured
`sources[]` array of `{n, title, domain, url}`. Both passes are billed, and
`run_rca` sums their usage into `result.usage` — the route settles credits from
it, so a pass omitted there is a pass nobody pays for.

**Known limitation:** grounding URLs are `vertexaisearch` redirects that expire
(~30 days). RCA results are not persisted, so this is tolerable; resolving to
canonical URLs would need an HTTP HEAD per source.

## Files

| Concern | File |
|---|---|
| Ontology (nodes/edges/vocab) | [app/domain/ontology.py](../app/domain/ontology.py) |
| Extraction (P6) | [app/pipeline/insight_extraction.py](../app/pipeline/insight_extraction.py) |
| Layer A persistence | [app/db/queries/insights.py](../app/db/queries/insights.py) |
| Projection (P7) | [app/pipeline/insight_projector.py](../app/pipeline/insight_projector.py) |
| Well/formation registry (gated) | [app/domain/well_registry.py](../app/domain/well_registry.py) |
| Rollup (P8) | [app/pipeline/insight_rollup.py](../app/pipeline/insight_rollup.py) |
| Shared fact queries | [app/db/queries/insight_facts.py](../app/db/queries/insight_facts.py) |
| Read + RCA routes | [app/routes/insights.py](../app/routes/insights.py) |
| RCA service | [app/pipeline/rca.py](../app/pipeline/rca.py) |
| RCA standards research (domain-restricted) | [app/pipeline/rca_research.py](../app/pipeline/rca_research.py) |
| Migration | `migrations/022_well_knowledge_layer.sql` / Alembic `011` |

## Config

| Setting | Default | Effect |
|---|---|---|
| `insight_extraction_enabled` | `True` | Master switch for Phases 6–8. Off = no rows written. |
| `insight_extraction_model` | `models/gemini-2.5-flash-lite` | Extraction model — cheap, run per part. |
| `insight_extraction_pages_per_part` | `5` | Pages per part (primary knob). Fewer = higher recall, more calls. |
| `insight_extraction_max_parts` | `60` | Safety cap on parallel LLM calls per document. |
| `insight_extraction_max_parallel` | `6` | Bounded concurrency for the parallel map. |
| `insight_extraction_max_chars` | `60000` | Per-**part** hard ceiling (one monster page). NOT whole-doc truncation. |
| `well_registry_enabled` | `False` | Off = well/formation FKs stay NULL, `fact_edges` skipped (see Scope). |
| `rca_web_search_enabled` | `True` | Master switch for the RCA research pass. Off = project evidence only (pre-2026-07-16 behaviour), one fewer LLM call, no search billing. |
| `rca_web_search_model` | `models/gemini-2.5-flash-lite` | Must support Search grounding. Billed **per search query executed**, on top of the call. |
| `rca_web_max_sources` | `8` | Cap on cited sources surviving the filter. |
| `rca_web_allowed_domains` | `""` (built-in) | Comma-separated override of `RCA_ALLOWED_DOMAINS`. Widening this widens what RCA will cite as authoritative. |
