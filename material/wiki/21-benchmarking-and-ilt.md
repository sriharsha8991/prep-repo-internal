# 21 — Benchmarking, Delta Attribution & Invisible Lost Time

Owns: how a well's elapsed time is compared against what the field has already demonstrated, how the gap is decomposed, and how that becomes an explanation with a page citation.

Planning documents live in [`docs/benchmarking/`](../docs/benchmarking/README.md) — one per requirement. This page owns the **connective tissue**: what exists, how the feature threads through the system, and which other wiki pages it touches.

---

## What this feature answers

> "Compared with the best performance this field has already achieved, is the current well ahead or behind — and if behind, exactly where did the time go?"

Not *"Well A took 40 days and Well B took 50."* The unit of comparison is an **activity inside a hole section inside a depth band**, never a depth alone.

## The four-layer shape

```
Historical DDRs  →  Best Composite  →  Live Well Comparison  →  Root Cause
```

| Layer | Requirement | Owns |
|---|---|---|
| Facts | FR-1, FR-2 | Activity timeline + normalisation |
| Benchmark | FR-3, FR-7, FR-8 | Composite cells, evidence grading, ladder level |
| Delta | FR-4, FR-11, FR-13 | Scope / NPT / ILT split, aftershock, confidence tiers |
| Explanation | FR-6, FR-10, FR-12 | RCA, response effectiveness, attention list |
| Access | FR-14, FR-15, FR-16 | Vector metadata, fact tool, filtered retrieval |
| Invariant | FR-5, FR-9 | Idempotent incremental update, determinism |

---

## What already exists today

**This is the important orientation fact: the destination is built; the PDF road to it is not.**

| Component | Where | State |
|---|---|---|
| `daily_reports` + `activity_intervals` tables | `app/db/models/ddr.py`, migration 025 | Shipped — see [#06](06-database-schema.md) |
| `DailyReport` / `Activity` + closed `ACTIVITY_CLASSES` | `app/domain/_ddr_model.py` | Shipped |
| `parse_witsml_ddr`, `merge_daily_reports` | `app/domain/ddr.py` | Shipped — deterministic, no LLM |
| Phase 7b DDR projector (idempotent) | `app/pipeline/ddr_projector.py` | Shipped — see [#03](03-ingestion-pipeline.md) |
| Benchmark reducer: curve, P50 / composite / best-actual / technical-limit | `app/pipeline/time_depth.py` | Shipped — pure, no I/O |
| `GET /projects/{id}/benchmark` | `app/routes/insights.py:892` | Shipped — see [#07](07-api-reference.md) |
| NPT closed vocabulary + `canonicalize()` | `app/domain/npt.py` | Shipped — see [#17](17-well-knowledge-layer.md) |

The gap is stated in the endpoint's own docstring: *"only the domain lane populates these today."* Nothing produces `DailyReport` objects from a **PDF**, and the reference corpus is 281 PDFs.

**FR-1 is therefore the unlock.** Once PDFs reach `daily_reports`, the projector, the reducer and the endpoint all light up unchanged.

---

## How it threads through the system

```
ingest PDF
  └─ Phase 0–4   markdown, chunks, vectors            → #03, #05
  └─ Phase 7b    ddr_projection
                   ├─ WITSML lane  parse_witsml_ddr   [exists]
                   └─ PDF lane     parse_pdf_ddr      [FR-1]
                        └─ daily_reports + activity_intervals   → #06
  └─ post-job    benchmark.recompute_project (debounced)        [FR-5]
                   └─ payload backfill onto Qdrant chunks       [FR-14] → #05

read paths (one shared query module: app/db/queries/benchmark_facts.py)
  ├─ GET /projects/{id}/benchmark          dashboard             → #07
  ├─ query_facts tool                      chat                  [FR-15] → #04
  └─ RCA evidence assembly                 pipeline/rca.py       [FR-6]  → #17
```

The shared query module is what structurally guarantees chat, dashboard and reports cannot disagree — the same rule [#17](17-well-knowledge-layer.md) already applies to NPT totals.

---

## Invariants this feature must not break

These are existing commitments, not new ones. Each has a documented failure behind it.

**No model computes an engineering quantity** (FR-9). `time_depth.py` is pure — *"the same inputs give the same curve every time, which is the whole point of a benchmark."* DDR parsing is structural. Layer B projection is pure SQL and free to re-run. Models do semantic normalisation and explanation only. See [#16](16-design-patterns.md).

**Never compute at query time.** Ingestion writes; a page load reads. This is what stops a chat answer and a dashboard disagreeing.

**Tenant scoping is enforced at the tool layer**, not the agent layer — *"an agent-layer filter can be argued out of position by a prompt injection in a document."* This is why FR-15 is a closed-surface query builder and not model-authored SQL. See [#02](02-auth-and-rbac.md), [#04](04-retrieval-and-chat.md).

**A benchmark is never set by a mis-parsed well.** `time_depth.WellCurve.is_benchmark_eligible` already enforces this; a flagged curve is reported but excluded from the benchmark set.

**A missed merge under-reports; a wrong merge lies.** `well_registry_enabled` stays False until the structural name parser lands (FR-2). See [#17](17-well-knowledge-layer.md).

**Do not add a tool that discourages `rag_search`.** Measured: three descriptions reading "PREFER THIS" stopped the model searching entirely. FR-15 *replaces* `fetch_project_insights` rather than joining it. See [#04](04-retrieval-and-chat.md).

---

## Things that are true about the data and easy to get wrong

Measured across the 281-DDR reference corpus, not sampled:

- **24-hour closure is a free correctness gate.** 257 of 281 reports close at exactly 24.00 h. Reports that do not balance are quarantined, never benchmarked. This is the only quality signal that needs no reference data, so it is what scales to customers we know nothing about.
- **The DDR carries two activity tables** — the 00:00–24:00 main table and a 00:00–06:00 next-morning table. Summing both gives a median of 30 h/day. Parse only the main one.
- **The NPT column needs word geometry, not reading order.** A naïve text parse finds 5 flagged rows out of 3,062; the column band is x ≈ 253–288 pt with comment text starting at x ≈ 290.
- **Section metadata is in only 23% of reports** and is skewed to later wells. Attribution must be derived through a four-rule chain.
- **There are no formation tops.** "Formation" is 100% present but almost entirely "Formation Imaging" (a logging tool). V1 proxies formation with depth band and blocks cross-field composites (FR-7).
- **The operator's plan baseline is unrecoverable.** `Job Days Ahead` (98.2% present) is re-baselined mid-well, so a plan curve cannot be reconstructed from it. Echo it as a scalar; let the composite lead.
- **Problem → action → outcome does not co-occur in a comment** — 702 problem mentions, 33 with an action, 2 with an outcome. It is a sequence over the timeline, not text mining (FR-10).
- **Units:** the tables are `_ft`; the corpus is metric. Convert once at ingest via `app/domain/units.py`. Do not add metric columns.

---

## Related pages

| Page | Relationship |
|---|---|
| [#03 Ingestion Pipeline](03-ingestion-pipeline.md) | Phase 7b is where FR-1 hooks in |
| [#04 Retrieval & Chat](04-retrieval-and-chat.md) | FR-15/FR-16 change the tool set and filters |
| [#05 Storage & Vectors](05-storage-and-vectors.md) | FR-14 extends the chunk payload |
| [#06 Database Schema](06-database-schema.md) | `daily_reports`, `activity_intervals`, new `source_tier` / `section_source` |
| [#07 API Reference](07-api-reference.md) | `GET /projects/{id}/benchmark` gains `cells`, `delta`, `attention`, `performance` |
| [#16 Design Patterns](16-design-patterns.md) | Pure-reducer and closed-vocabulary patterns this feature leans on |
| [#17 Well Knowledge Layer](17-well-knowledge-layer.md) | Shares the fact tables, the NPT vocabulary and the RCA path |

Frontend companions: [`frontend_docs/TIME_DEPTH_BENCHMARK_PLAN.md`](../frontend_docs/TIME_DEPTH_BENCHMARK_PLAN.md) and [`DDR_PDF_TIME_DEPTH_PLAN.md`](../frontend_docs/DDR_PDF_TIME_DEPTH_PLAN.md) — earlier plans covering the v1 curve. Where they disagree with `docs/benchmarking/`, the FRD is newer; reconcile rather than assuming either.

---

## Status

Pre-implementation. Nothing in `docs/benchmarking/` is shipped except the components listed under *What already exists*. Update this section as requirements land, and date the change per the wiki's own convention.
