# Wellsynthai Backend Wiki

This wiki is the engineering source-of-truth for the Wellsynthai backend.
Each page owns one slice of the system. If you update behaviour, update
the page that owns it — do not duplicate facts across pages.

## Pages

| # | Page | Owns |
|---|---|---|
| 01 | [Architecture](01-architecture.md) | Service topology, request lifecycle, where things live |
| 02 | [Auth & RBAC](02-auth-and-rbac.md) | Tiers, JWT verification, invite flow, quotas, error codes |
| 03 | [Ingestion Pipeline](03-ingestion-pipeline.md) | PDF → markdown/images/tables/entities; Celery topology |
| 04 | [Retrieval & Chat](04-retrieval-and-chat.md) | RAG agent, tools, tier weighting, image grounding, soft-land |
| 05 | [Storage & Vectors](05-storage-and-vectors.md) | Local filesystem buckets + Qdrant collections + tenant payload |
| 06 | [Database Schema](06-database-schema.md) | Postgres tables, enums, triggers (local Postgres, no RLS) |
| 07 | [API Reference](07-api-reference.md) | Endpoint-by-endpoint contract; canonical request/response |
| 08 | [Deployment & Ops](08-deployment-and-ops.md) | Docker stack, env, scaling knobs, log channels |
| 09 | [Testing](09-testing.md) | Smoke + unit; how to add a new test |
| 10 | [Runbooks](10-runbooks.md) | Bootstrap super_admin, recover lost password, reset Qdrant, replay an ingest |
| 11 | [Memory + ChatGPT-feel Roadmap](11-memory-and-chatgpt-roadmap.md) | Backend plan: STM/LTM, streaming, additional synth tools, verifier |
| 12 | [Frontend Implementation Plan](12-frontend-implementation-plan.md) | FE plan companion to #11: streaming UI, memory drawer, chips, slash commands |
| 13 | [Document Viewer & Serving](13-document-viewer-and-serving.md) | Per-page serving endpoints + lazy/virtualized viewer contract for large PDFs |
| 14 | [Billing, Demo Tier & Feedback](14-billing-demo-and-feedback.md) | Token-budget credit system, public demo self-signup + email verification + IP gating, feedback-for-tokens, book-a-demo |
| 15 | [Report Generation](15-report-generation.md) | Templates + run lifecycle; legacy single-shot vs agentic master+per-section LangGraph path (behind `report_agentic_enabled`) |
| 16 | [Design Patterns](16-design-patterns.md) | Recurring architectural + GoF patterns the backend commits to, where each lives, and what we deliberately avoid |
| 17 | [Well Knowledge Layer](17-well-knowledge-layer.md) | Ingestion-time NPT + Lessons extraction, top-5 rollups, and the insights/RCA APIs |
| 18 | [Credits & Billing, Explained](18-credits-and-billing-explained.md) | Plain-English walkthrough of the credit/billing system — companion to #14 |
| 19 | [User & Org Management, Explained](19-user-and-org-management-explained.md) | Plain-English walkthrough of roles, invites, and organization management — companion to #02 |
| 20 | [Marketing Blog](20-blog.md) | Public anonymous blog + super_admin authoring; the media upload→storage→serving pipeline and its per-environment URL config |
| 21 | [Benchmarking & ILT](21-benchmarking-and-ilt.md) | Time–depth benchmarking from DDRs: best-composite curve, scope/NPT/ILT decomposition, RCA. Per-requirement plans in [`docs/benchmarking/`](../docs/benchmarking/README.md) |
| 22 | [TLS / HTTPS](22-tls-and-https.md) | Let's Encrypt certificate for `api.wellsynth.ai`: webroot renewal, GoDaddy DNS, rolling nginx/TLS changes onto the prod VM via Portainer |

## Stack note (2026 migration)

The backend migrated off Supabase to a self-hosted Docker stack: **local
Postgres** (not Supabase Postgres), **local filesystem storage** (not Supabase
Storage / signed URLs), and **self-hosted JWT (HS256) + bcrypt** auth (not
Supabase Auth / JWKS). RLS and Supabase Realtime were dropped — ownership is
enforced in app code; the FE polls job status. Planning docs:
[INGESTION_FLOW.md](../data/plan_documentations/INGESTION_FLOW.md), [PLAN_pdf_sharding.md](../data/plan_documentations/PLAN_pdf_sharding.md).

## How this wiki is maintained

- **One page = one responsibility.** If a page is tempted to explain
  something owned by another page, link to that page instead of
  copy-pasting.
- **Code is the source of truth, not the wiki.** The wiki explains
  *why* and *how to operate*. For *what exactly the code does*, link to
  the file/line and let the reader read it.
- **Date every behaviour change.** When you change a contract, prepend
  the section's first paragraph with `> _Changed YYYY-MM-DD: ..._`.
- **Wiki edits ship in the same PR as the code change.** A code change
  that contradicts the wiki is a regression.

## Where this wiki ends and other docs begin

- `README.md` (repo root) — quickstart for new contributors. Stays short.
- `frontend_docs/communication_from_backend.md` — backend → frontend
  contract. **Authoritative for the FE.** This wiki must not contradict
  it; if something here disagrees, the FE doc wins and we open a wiki
  issue to reconcile.
- `BACKEND_SPEC.md`, `PLAN*.md`, `DEV_LOG.md` — historical / planning.
  Do not treat as live.
- `.github/copilot-instructions.md` — project guardrails for AI agents
  and humans alike. Read once on day one.
