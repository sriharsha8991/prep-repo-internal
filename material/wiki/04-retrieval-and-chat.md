# 04 — Retrieval & Chat

> _Changed 2026-06-07: corrected model names, agent shape, reranker, and
> image-serving for the local migration. Owns: how `/chat` and `/search` turn
> a query into a grounded answer, including tier weighting, tenant filtering, and
> image evidence._

## Live request path

```
POST /chat {message, session_id?, project_id?, document_id?, temp?}
  └─ middleware → AuthUser
  └─ rag agent (2-layer: Guard → Synthesizer)
        ├─ Guard (gemini-2.5-flash-lite): route / meta-answer / decide retrieval
        ├─ search_chunks(...)   ─┐
        ├─ search_entities(...) ─┼─ all calls carry the user_id Qdrant filter
        ├─ search_images(...)   ─┘
        ├─ tier weighting (artifact-class; no cross-encoder)
        └─ Synthesizer (gemini-3.5-flash): grounded answer + citations (multimodal)
  └─ {answer, sources[], evidence[], claims[], unanswered[],
      attached_pages[], session_id, meta}
```

`POST /chat/stream` is the same contract over **SSE** (progressive answer +
final `done` frame carrying `meta`/`sources`/`evidence`). Persisted turns live
under `/chat/sessions/*` — see [07-api-reference.md](07-api-reference.md).

Models come from [app/config.py](../app/config.py): `chat_triage_model` /
`chat_lite_model` = `gemini-2.5-flash-lite`, `chat_synth_model` =
`gemini-3.5-flash`. Embeddings = `gemini-embedding-001` (768-d).

## RAG agent

[app/agents/rag/](../app/agents/) (agent + tools), [app/agents/prompts.py](../app/agents/prompts.py)

- It is a **2-layer orchestrator** (Guard → Synthesizer), not a planner/executor
  loop and not the per-page LangGraph in [graph.py](../app/agents/graph.py)
  (that graph belongs to ingestion extraction, not chat).
- Tools are deliberately few:
  - `search_chunks` (text)
  - `search_entities` (formations / units / wells / equipment)
  - `search_images` (per-page renderings)
- Each tool **must** accept `user_id` and pass it as a Qdrant filter.
  Cross-user leakage is a security bug, not a polish item.

## Evidence-tier weighting (not a reranker)

[app/retrieval/tier_weighting.py](../app/retrieval/tier_weighting.py)

- **No semantic reranking happens anywhere in this pipeline.** There is no
  cross-encoder and no second model call; query↔passage relevance is never
  recomputed after the Qdrant search.
- What it does: weights each hit's raw similarity by a static **artifact-class
  tier weight** (`adjusted_score = raw_score * tier_weight(artifact_class)`),
  so authoritative document classes outrank incidental ones. Ties keep
  Qdrant's incoming similarity order (stable sort).
- A missing/unknown `artifact_class` scores the 0.5 default. When *no* result
  carries a class the weighting is a no-op on ordering; in a *mixed* set an
  unclassified document is penalized 50% against a classified Tier-1 one.
- Top hits plus image evidence are passed to the Synthesizer prompt.
- Adding a real cross-encoder reranker is tracked in
  [PROVIDER_OPTIONS.md](../PROVIDER_OPTIONS.md) as the highest quality-per-effort
  retrieval change available.

## Image evidence and `attached_pages`

The Synthesizer is multimodal — when an image point is weighted into the
top set, the agent attaches the page PNG as inline image input. The response
surfaces these in `attached_pages[]` with `{document_id, page_num, url}` so FE
can render thumbnails.

URLs are `/documents/{id}/pages/{n}/image`, which returns the **image bytes
directly** (200 OK, optional `?w=` downscale / WebP), not a 302 redirect to a
signed URL. See [13-document-viewer-and-serving.md](13-document-viewer-and-serving.md).

## Tenant filtering (mandatory)

Every Qdrant query carries:

```python
filter = Filter(
    must=[
        FieldCondition(key="user_id", match=MatchValue(value=auth_user.id)),
        # plus optional project_id / document_id
    ]
)
```

This is enforced at the tool layer, not the agent layer, so an agent
prompt-injection attack cannot bypass it.

For super_admin support reads, we still **do not** disable the filter
silently. Either super_admin uses an explicit `?scope=all` parameter on
list endpoints (currently project list and admin/users list), or the
read goes through Postgres directly. Vector search is always scoped to
the calling user.

## Soft-land on empty corpus

When tool calls return zero hits (new account, fresh project,
mismatched scope), the agent does not hallucinate. It returns a
deterministic answer:

> "I don't have any documents indexed yet under your account. Once
> you ingest a PDF I can answer questions grounded in it."

This is what allows our smoke test to call `/chat` against an empty
super_admin corpus and still get a 200.

## Search endpoint

`GET /search?q=...&project_id=...&document_id=...&limit=10`

- Direct semantic search bypassing the agent. Returns top chunks with
  `{score, text, document_id, page_num, source_path}`.
- Same `user_id` filter applies.
- `q` is required. Bogus role values etc. → 422.

## Chat request body

`POST /chat`:

```json
{
  "message": "What is the spud date of well A?",
  "session_id": "optional uuid — thread into an existing conversation",
  "project_id": "optional uuid scope",
  "document_id": "optional uuid scope",
  "temp": false
}
```

Note: the field is `message`, not `query`. Earlier doc revisions used
`query`; the live API has been `message` since PR-7. Omit `session_id` to start
a new conversation (the response echoes the new id). `temp: true` is an
incognito turn — no long-term memory capture, 1-hour Redis session TTL.

## Chat response shape

```json
{
  "answer": "...",
  "sources": [
    {"source_id": "...", "text": "...", "score": 0.83,
     "doc_name": "...", "page_num": 4, "kind": "chunk", "collection": "..."}
  ],
  "evidence": [
    {"kind": "table|bbox_crop|page_image", "doc_name": "...",
     "page_num": 4, "url": "..."}
  ],
  "claims": [{"text": "...", "source_id": "...", "confidence": "medium"}],
  "unanswered": [{"facet": "...", "reason": "..."}],
  "attached_pages": [
    {"document_id": "...", "page_num": 4, "url": "/documents/.../pages/4/image"}
  ],
  "session_id": "...",
  "meta": { }
}
```

Every `sources[i]` and `evidence[i]` carries `document_id`. FE uses
this for citation links and image rendering. Do not use `job_id` for
links — it is per-ingestion-run only.

## Things this page does not own

- The model identifiers and pricing → [app/config.py](../app/config.py) +
  [08-deployment-and-ops.md](08-deployment-and-ops.md).
- Where the chunks came from → [03-ingestion-pipeline.md](03-ingestion-pipeline.md).
- Qdrant collection schema → [05-storage-and-vectors.md](05-storage-and-vectors.md).
