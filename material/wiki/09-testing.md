# 09 — Testing

> _Changed 2026-06-07: removed the stale `e2e_*` layer (those tools no longer
> exist) and Supabase references. Owns: every kind of test in the repo and how
> to run/extend it._

## Test layers

| Layer | Where | Purpose |
|---|---|---|
| Unit | `tests/test_*.py` (pytest) | Pure-python tests for pipeline + agents + routes (stubbed Gemini, no network) |
| API smoke | `app/tools/api_smoke.py` | Fast contract check across the FE-facing endpoints against a live stack |

There is no separate `e2e_*` layer anymore — negative/role-gating cases live in
the unit suite or the smoke script.

## Unit tests (pytest)

```powershell
.\.venv\Scripts\python.exe -m pytest tests/ -q
```

> Note: `pytest` is **not** installed in the runtime Docker image — run the
> suite from the dev venv, not inside the `wellsynthai-app` container.

Current files (`tests/`):
- `test_agents.py` — extractor (self-validating) + escalation with stubbed Gemini.
- `test_page_streamer.py` — rasterizer concurrency + streaming semantics.
- `test_preprocessing.py` — fitz baseline + liteparse edge cases.
- `test_postprocessor.py` — full-document aggregation + cross-page table merge.
- `test_classifier.py` / `test_artifact_classifier.py` / `test_ingest_artifact_class.py` — page + artifact-class classification.
- `test_entity_normalizer_and_guardrail.py` — controlled vocab + numeric guardrails.
- `test_tier_weighting_governance.py` — artifact-class tier weighting.
- `test_chat_sessions.py`, `test_slash.py`, `test_memory_temp_outline.py` — chat sessions / slash commands / memory.
- `test_support_tickets.py`, `test_support_admin.py` — support flow.
- `test_credits.py` — reserve/settle math, credit re-denomination (1 credit = 100k tokens), paid-org soft-cap (never 402) + org spend-alert odometer (1×/2×/3× tiers fire once each, reset at renewal, per-org threshold override, kill switch).
- `conftest.py` — shared fixtures (e.g. sample PDF).

When you add a feature, add a unit test under the relevant file; don't introduce
new files unless the topic is genuinely new.

## API smoke

[app/tools/api_smoke.py](../app/tools/api_smoke.py) logs in as the bootstrapped
super_admin and exercises the FE-facing surface (auth good/bad, `/auth/me`,
profile round-trip, refresh, admin users list/invite, projects CRUD, documents
list + 404, ingest list + 404, search on empty corpus, chat soft-land 200).

```powershell
.\.venv\Scripts\python.exe -m app.tools.api_smoke
```

Expect `failed: 0`. Run this first after any change touching routing,
middleware, or config.

## Other dev tools (`app/tools/`)
- `bootstrap_super_admin.py` — create/promote a super_admin (see [10](10-runbooks.md)).
- `qdrant_reset.py` — wipe + recreate the org-scoped collection set.
- `dump_openapi.py` — regenerate `openapi.json`.
- `probe_chat.py` — manual chat probe against a live stack.

## Email in tests
Smoke/dev scripts use `example.com` addresses (Resend rejects → backend rolls
back, the script accepts the resulting error). The full mail round-trip is a
manual test against your own inbox — see [10-runbooks.md](10-runbooks.md).

## Things this page does not own
- The actual contract being tested → [07-api-reference.md](07-api-reference.md).
- How to fix a failing test → debugging, not testing.
