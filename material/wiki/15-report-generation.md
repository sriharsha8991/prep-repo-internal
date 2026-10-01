# 15 — Report Generation

> _Owns: how strategic reports are generated — templates, the run lifecycle, the
> legacy single-shot path, and the agentic master+planner path._
> _Added 2026-06-14. Source of truth: [app/agents/reports/](../app/agents/reports/),
> [app/routes/reports.py](../app/routes/reports.py), [app/workers/tasks/report.py](../app/workers/tasks/report.py),
> design doc [PLAN_v7_report_agent.md](../data/plan_documentations/PLAN_v7_report_agent.md)._

## What a report is

A report is a `ReportTemplateSpec` of ordered `SectionSpec`s
([templates.py](../app/agents/reports/templates.py)). Built-ins: `well_strategic_v1`
(10 sections) and `single_well_eowr_v1` (5). Custom templates are generated from a
plain-English brief ([template_generator.py](../app/agents/reports/template_generator.py))
and stored in `report_templates`; resolved via `resolve_template`
([registry.py](../app/agents/reports/registry.py)).

`SectionSpec` core fields: `number, title, probes (retrieval queries), task_prompt
(writing instruction), depends_on_sections, chunk/entity_limit`. Agentic-path
fields (optional): `decision_objective, lenses, sub_question_seeds, rubric`.

## Run lifecycle (unchanged across both paths)

1. `POST /projects/{id}/reports` ([reports.py](../app/routes/reports.py)) — access
   check, one-in-flight guard, **reserve credits** (token estimate), create the
   `strategic_reports` row (`status=queued`), dispatch the Celery task.
2. The `report.generate` Celery task ([report.py](../app/workers/tasks/report.py))
   builds embedder/vector-store/`genai.Client`, resolves org/collection scope, and
   calls `run_report_generation` ([orchestrator.py](../app/agents/reports/orchestrator.py)).
3. Orchestrator: load docs → compute deterministic metrics
   ([metrics.py](../app/agents/reports/metrics.py)) → **generate sections** (path
   below) → assemble markdown (+ coverage appendix + data-gap register) →
   persist (`status=succeeded`, `markdown`, `sections_meta`) → record usage +
   **settle credits on measured tokens**. The FE polls the row
   (`status`, `current_phase`, `sections_done`).

## Two generation paths (flag: `report_agentic_enabled`)

### Legacy (flag OFF, default) — single-shot
`builder.build_section` ([builder.py](../app/agents/reports/builder.py)): per section,
pre-fetch evidence (`retrieve_for_section`), serialize (capped/truncated in
[serialize.py](../app/agents/reports/serialize.py)), **one** `generate_content` call
on the cheap `report_synth_model`. Independent sections run in parallel; dependent
sections (recommendations/lessons) synthesize from upstream markdown only — no
retrieval. Fast/cheap; thinner, and prone to repetition + zero-hit dependent
sections.

### Agentic (flag ON) — master orchestrator + per-section LangGraph analysts
See [PLAN_v7_report_agent.md](../data/plan_documentations/PLAN_v7_report_agent.md) for rationale and the
architecture review (STORM / Self-RAG / CRAG / agentic-RAG).

- **Master** ([master.py](../app/agents/reports/master.py), top reasoning model):
  derives the report's **decision-agenda** + a per-section **topic-ownership map**
  (prevents repetition up front), delegates to N section agents, curates the
  **blackboard** (sole writer; compact digests, not raw evidence), reviews each
  section once against the blackboard (one targeted re-run), then **composes** the
  final clean report (citations preserved, no new claims).
- **Section analyst** ([section_agent.py](../app/agents/reports/section_agent.py) +
  [agent_nodes.py](../app/agents/reports/agent_nodes.py)) — a compiled LangGraph per
  section: `plan → gather → (skip|reason) → draft → reflect → (revise≤2|END)`.
  Retrieval-as-a-tool ([retrieval_tool.py](../app/agents/reports/retrieval_tool.py),
  function-calling loop over `rag_search`) gathers on demand with **no evidence
  caps** ([evidence_memory.py](../app/agents/reports/evidence_memory.py), map-reduce
  condensation that preserves numbers + citations). Dependent sections **also
  retrieve** their own evidence (fixes the zero-hit bug).
- **Bounded loops:** ≤2 section self-reflections + 1 master review per section.
- **Scales to N (e.g. 10):** sections fan out (capped by
  `report_max_concurrent_sections_per_worker`); the master reasons over digests so
  its context stays ~constant in N; compose is map-reduce.

## Config knobs ([config.py](../app/config.py))

```
report_agentic_enabled=False           # master switch (legacy is the rollback)
report_master_model=models/gemini-3.5-flash      report_master_thinking=high
report_planner_model=models/gemini-3.5-flash   report_planner_thinking=medium
report_grader_model=models/gemini-2.5-flash-lite
report_section_max_reflections=2       report_master_review_loops=1
report_retrieval_max_calls=12          report_web_search_enabled=False
report_agent_max_tokens=16384          report_master_compose_max_tokens=32768
report_synth_model / report_synth_max_tokens=16384   # legacy path
report_max_concurrent_sections_per_worker=10
```

Cost/latency are materially higher on the agentic path (top-tier master + N
analysts + reflection) — by design; quality first.

## Persistence (agentic extras)

`strategic_reports` (see [06-database-schema.md](06-database-schema.md)) gains
`blackboard JSONB` (agenda, ownership map, section digests, review notes, compose
log) and `quality_score JSONB` (per-section rubric scores), written only on the
agentic path (migration `008_report_blackboard.sql`). The FE contract for these is
in [§0.13 of communication_from_backend.md](../frontend_docs/communication_from_backend.md).

## Downloads (PDF / DOCX)

Both `GET .../reports/{id}/pdf` and `GET .../reports/{id}/docx` render on demand
from the stored markdown ([reports.py](../app/routes/reports.py)). The markdown
embeds figures as **relative, auth-gated** URLs
(`![...](/documents/{doc_id}/pages/{n}/image)`, emitted by
[evidence_memory.py](../app/agents/reports/evidence_memory.py)) — the FE resolves
those against the authenticated origin, so they only render **on screen**. Offline
renderers can't follow them, so before rendering, both routes call
`resolve_report_images` ([report_images.py](../app/agents/reports/report_images.py)),
which reads each referenced page PNG from storage (scoped to the report's project),
downscales it, and rewrites the ref to an inline `data:` URI. Unresolvable refs
degrade to their caption text.

- PDF: markdown → HTML → PDF via weasyprint ([pdf_renderer.py](../app/agents/reports/pdf_renderer.py)).
- DOCX: markdown → HTML → Word via python-docx ([docx_renderer.py](../app/agents/reports/docx_renderer.py));
  handles headings, inline formatting, lists, GFM tables, and the inlined images.

## Things this page does not own
- Endpoint request/response shapes → [07-api-reference.md](07-api-reference.md).
- `strategic_reports` / `report_templates` columns → [06-database-schema.md](06-database-schema.md).
- Underlying retrieval (rag_search, tier weighting, governance) → [04-retrieval-and-chat.md](04-retrieval-and-chat.md).
- Credit reserve/settle → [14-billing-demo-and-feedback.md](14-billing-demo-and-feedback.md).
