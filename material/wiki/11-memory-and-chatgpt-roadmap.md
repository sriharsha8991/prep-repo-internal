# Wellsynthai — Backend Implementation Plan: Memory + ChatGPT-feel Roadmap

> ⚠️ **Pre-migration roadmap (2026-06-07).** This plan was written before the
> Supabase→local migration. Where it says "Supabase Postgres / Storage / JWKS /
> `auth.users` / `supabase/migrations/`", read **local Postgres / local
> filesystem / HS256 JWT / `users_profile` / `migrations/`**, and ignore RLS
> (ownership is app-level now). The memory *feature design* still stands; only
> the persistence/auth substrate changed. See pages 01–06 for current state.
>
> **Scope of this document.** Backend-only implementation plan covering
> (1) short-term + long-term user memory and (2) the prioritized roadmap
> of additional synthesizer tools / UX upgrades to make the chat feel
> like ChatGPT.
>
> Frontend changes live in a separate document: `12-frontend-implementation-plan.md`.
>
> **Guardrail discipline.** Nothing in this plan adds a feature without a
> matching user request. Implement items by name, in the recommended
> order. Don't roll a tier in one go.

---

## 0. Recap of where we are

- **Chat pipeline (live).** 2 layers: regex prefilter → guard LLM (Flash, JSON) → synthesizer LLM (Flash, function-calling on `rag_search`, max 3 calls, top-10 sources).
- **Retrieval (live).** Single `rag_search()` fan-out across `chunks`, `entities` and (conditionally) `images`; `±1 chunk_index` neighbor expansion on the top-3 narrative chunks.
- **Persistence (live).** Supabase Postgres for `documents`, `jobs`, `briefings`, `lessons`, `pinboards`, `user_profile`. Auth via Supabase JWKS. Storage in `pdfs / pages / artifacts` buckets. Redis is Celery broker + short-lived progress cache.
- **Memory (today).** In-process `ChatSession` per `session_id`; lost on worker restart, not shared across workers. No long-term recall.

---

## 1. Recommended implementation order

| # | Item | Tier | Why first |
|---|---|---|---|
| 1 | **Streaming `POST /chat/stream`** (SSE) | UX 1 | Single biggest perceived quality jump, smallest blast radius. No new tools. |
| 2 | **Short-term memory hardening** (Redis-backed session, rolling summary, sticky entities) | Memory STM | Removes worker-restart amnesia and bounds context window before long-term memory lands. |
| 3 | **Long-term `user_memories` table + passive capture + recall** | Memory LTM | "This assistant knows me" inflection point. Done after STM so we don't double-process. |
| 4 | **`fetch_document_outline` tool** | Tier 1.2 | Fixes the "tell me about this document" class cheaply, without burning RAG calls. |
| 5 | **Follow-up suggestion chips** (already in the contract — surface 2–3) | Tier 3.9 | Free quality lift; FE work is small. |
| 6 | **`fetch_table` tool** | Tier 1.3 | Removes table-truncation pain on tabular follow-ups. |
| 7 | **`compare_documents` tool** | Tier 1.4 | Replaces ad-hoc multi-doc handling with a typed call. |
| 8 | **Source verifier post-pass** | Tier 2.7 | Strikes through unverified `[Sn]` claims; restores the validator we removed, but as a *post*-step. |
| 9 | **Slash-commands** (`/scope`, `/clear`, `/forget`, `/help`) | Tier 3.8 | Free, deterministic, no LLM cost. |

**Out of scope** for this plan: voice I/O, web browsing, code interpreter, image generation, plugin store, agentic browsing.

---

## 2. Item details

### 2.1 Streaming chat — `POST /chat/stream` (SSE)

**Goal.** Token-by-token assistant output appears as the synthesizer generates it. The chat already returns a single `text` from `model.aio.generate_content`; switch the synthesizer's *final* generation pass to a stream and tunnel it through Server-Sent Events.

**Files touched.**

- `app/routes/chat.py` — add a sibling endpoint `POST /chat/stream`. Same body, returns `text/event-stream`. Existing `POST /chat` stays for non-streaming clients (smoke tests, internal tools).
- `app/agents/rag/agent.py` — add `run_agent_stream(...)` that mirrors `run_agent` but yields events:
  ```
  event: status   data: {"phase":"guard"}
  event: status   data: {"phase":"rag_search","query":"..."}
  event: token    data: "Yes,"
  event: token    data: " the 9-5/8\""
  ...
  event: done     data: {"sources":[...], "attached_pages":[...], "tool_calls_made":2, "meta":{...}}
  ```
- `app/agents/rag/synthesizer.py` — split into two paths:
  - `synthesize(...)` (current) — returns `AgentResponse`.
  - `synthesize_stream(...)` — async-generator yielding `("status"|"token", payload)`. The function-calling loop is identical until the *final* model turn; that final call uses `client.aio.models.generate_content_stream(...)`. Tool-call turns are not streamed back to the client (only the final answer text is); we emit `event: status` with `phase=rag_search` so the UI can show "Searching documents…".

**Guard's role in streaming.** The guard's `reply` and `clarify` outputs are short text and arrive in one shot — we still emit them via SSE for protocol uniformity (`event: token` then `event: done`).

**Cancellation.** SSE clients disconnect → uvicorn raises `ClientDisconnect`. Catch in the route and `task.cancel()` the in-flight Gemini generator. Costs are low but worth doing.

**Backpressure / reconnection.** Out of scope. Best-effort SSE only; if a connection drops, the user retries.

**Auth.** `/chat/stream` reuses the same `Depends(current_user)` middleware; the bearer token is passed in the `Authorization` header. We do *not* support cookie auth or query-string tokens (logs would leak them).

**Wire-format example (one consolidated stream).**
```
data: {"event":"status","phase":"guard"}

data: {"event":"status","phase":"rag_search","query":"NPT Sajaa 6"}

data: {"event":"token","text":"Yes, "}
data: {"event":"token","text":"NPT for Sajaa 6 was "}
...

data: {"event":"done","sources":[...],"attached_pages":[...],"meta":{...}}
```
(We use the simpler `data:`-only shape because EventSource clients with custom `event:` lines need explicit listeners. One JSON-per-message is more portable.)

**Test plan.** Add a smoke step that opens the SSE stream, reads N events, confirms a `done` event arrives, and the union of `token` events matches the non-streaming `/chat` answer modulo whitespace.

**Estimated LOC.** ~150 backend, ~40 in `api_smoke.py`.

---

### 2.2 Short-term memory hardening (Redis-backed sessions)

**Problems today.**

1. `ChatSession` is `defaultdict` in process memory → lost on container restart, not shared across uvicorn workers.
2. History grows linearly (`MAX_HISTORY_TURNS * 2 = 20` messages). At 80 words/message that's ~1600 tokens added to every guard call.
3. Coreference resolution depends on the model spotting the right anchor in raw history.

**Plan.**

#### A. Move `ChatSession` to Redis

- Key: `chat:session:{session_id}`. Value: msgpack-encoded dict with `messages`, `last_sources` (only `source_id` + minimal metadata, not full `SearchResult`), `summary`, `current_subject`.
- TTL: 24h sliding (refreshed on every read/write).
- API: `app/agents/rag/sessions.py` (new file) exposing `get_session_async(session_id) -> ChatSession` and `save_session_async(session_id, session)`. `agent.py` and `guard.py` call these instead of the current `defaultdict`.
- The existing `ChatSession` dataclass keeps its shape; only the storage swaps.

**Reuses existing Redis** (Celery broker connection from `settings.celery_broker_url`). No new dependency, no new container.

#### B. Rolling summary

- Trigger: every time the session reaches `MAX_HISTORY_TURNS = 12` messages, compress the oldest 6 into one paragraph (≤120 tokens) via Flash and replace those 6 messages with one synthetic `system: <summary>` entry.
- The summary is regenerated, not appended — we never let a summary chain explode.
- Cost: 1 cheap Flash call every ~6 turns; net token savings on the guard call from turn 13 onward.

#### C. Sticky entities — `current_subject`

- After every `answered` turn, post-process the assistant reply with a tiny regex pass + a 1-line Flash extractor: `"From this answer, what is the primary subject? Reply as JSON: {well, document, formation, depth_datum, run}."`
- Store on the session as `current_subject`. Truncate values to 80 chars; drop nulls.
- The guard's prompt prepends a single line: `Current subject (carry over to follow-ups unless the user changes it): well=Sajaa 6; document=Sajaa 6 EOWR; depth_datum=KB.`
- Result: "the deeper one" always disambiguates correctly even when history is summarised away.

#### D. What changes in code?

- New file: `app/agents/rag/sessions.py` (Redis-backed I/O).
- Update `app/agents/rag/types.py`: `ChatSession` gets `summary: str` and `current_subject: dict[str,str]` fields.
- Update `app/agents/rag/agent.py`: `run_agent` gets `await get_session_async(session_id)` and `await save_session_async(...)` at end. The history-text builder uses `summary + last 6 messages`.
- Update `app/agents/rag/guard.py`: prompt block adds `current_subject` line.
- Add a Celery task (or simple in-request inline call) `summarize_session_tail(session_id)` — fired only when history hits the cap.

**Estimated LOC.** ~250 backend.

---

### 2.3 Long-term memory — `user_memories` (cross-session, per-user)

#### Schema (one Supabase migration)

```sql
create table public.user_memories (
  user_id      uuid not null references auth.users(id) on delete cascade,
  scope        text not null check (scope in ('preference','fact','project_pin','style','dismissed')),
  key          text not null,
  value        jsonb not null,
  source_msg   text,
  confidence   real not null default 0.5 check (confidence between 0 and 1),
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  primary key (user_id, scope, key)
);

create index user_memories_user_idx on public.user_memories(user_id, scope);
```

Migration file: `supabase/migrations/<ts>_add_user_memories.sql`.

#### Capture — passive, post-turn

- New Celery task `extract_user_memory(user_id, user_msg, assistant_msg, session_id)` in `app/workers/tasks/memory.py`.
- Fires only when **all** of: status was `answered`; user message is `> 6 words`; assistant answer is `> 60 words`.
- Calls Flash once with this prompt:
  ```
  Extract durable items the assistant should remember about THIS user.
  Categories:
    - preference: e.g. "always uses metric units", "prefers concise replies"
    - fact:       e.g. "is a drilling engineer at Petrobras"
    - project_pin:e.g. "default project is Sajaa"
    - style:      tone/length/formality cues
  Output JSON list. Return [] if nothing durable.
  ```
- Upsert per `(user_id, scope, key)`. On collision: bump `confidence = min(1, old + 0.1)` and update `value` if the new value differs. On contradiction: keep newer, log `memory_contradicted`.
- **Drop low-signal:** entries with `confidence < 0.5` after first insert are not surfaced to the guard until they cross the threshold.

#### Recall — load on every chat turn

- `app/db/queries/context_card.py::build_context_card()` gets a new `memories` block:
  ```python
  card["memories"] = await fetch_top_memories(
      user_id=user_id, max_items=8, min_confidence=0.5
  )
  # Already-bounded snapshot; ranks by confidence × recency.
  ```
- The guard's system prompt remains unchanged (already says "use the CONTEXT CARD"); the new block surfaces in the user message naturally.
- Total tokens added per turn: ≤ 400. Bounded.

#### Style profile — single record per user

- `scope='style', key='profile'`. Value: `{ "avg_msg_len": 42, "formality": "neutral", "preferred_shape": "terse_technical" }`.
- Updated by the same Celery task, but only every Nth call (e.g. every 5 turns) to avoid churn.
- The synthesizer's system prompt gets one extra line at the bottom (read from the card): `Style hint: user prefers terse, technical answers.` Defaults to nothing if absent.

#### Privacy + control endpoints

- `GET    /memories?scope=...` — list. Owner-scoped.
- `DELETE /memories/{scope}/{key}` — soft-delete (sets `scope='dismissed'`); 30-day grace before vacuum.
- `POST   /memories/forget-all` — bulk soft-delete.
- All three live in a new file `app/routes/memories.py` and route through standard `Depends(current_user)`.
- Delete operations always log `memory_deleted` for auditability.

#### Files

- `supabase/migrations/<ts>_add_user_memories.sql`
- `app/db/models.py` — add `UserMemory` SQLAlchemy model.
- `app/db/queries/memories.py` — `fetch_top_memories`, `upsert_memory`, `forget_*`.
- `app/db/queries/context_card.py` — wire the `memories` block.
- `app/workers/tasks/memory.py` — Celery task.
- `app/agents/rag/agent.py` — fire-and-forget enqueue at end of `answered` turns.
- `app/routes/memories.py` — three endpoints.
- `app/main.py` — `app.include_router(memories.router)`.

**Estimated LOC.** ~600 backend + 1 SQL migration.

---

### 2.4 `fetch_document_outline(document_id)` synthesizer tool

**Why.** *"Tell me about this document"* burns RAG calls today; the answer doesn't need similarity search at all — it needs the document's structure.

**Implementation.**

- New helper `app/db/queries/document_outline.py::fetch_outline(document_id, user_id)`.
  - Returns:
    ```python
    {
        "filename": str,
        "page_count": int,
        "headings": [{"page": int, "text": str}, ...],   # from artifacts/<id>/headings.json
        "table_count": int,
        "image_count": int,
        "ingested_at": iso,
        "sample_pages": [1, page_count // 2, page_count],  # for quick lookup
    }
    ```
- Source: read from existing per-document artifacts written by the postprocessor. If `headings.json` doesn't exist for older docs, derive from `full_document.md` first-pass H1/H2 lines.
- New tool declaration in `app/agents/rag/synthesizer.py`:
  ```python
  fetch_document_outline_tool = types.FunctionDeclaration(
      name="fetch_document_outline",
      description=(
          "Get the high-level shape of a single ingested document — page count, "
          "headings, table count, image count. Use for 'tell me about this "
          "document', 'what's in document X', 'how long is this'. Do NOT use "
          "for content-detail questions."
      ),
      parameters=...,  # required: document_id (string)
  )
  ```
  Synthesizer is allowed `rag_search` AND `fetch_document_outline` in the same turn (still capped at 3 total calls combined).

- Tool dispatch in the synth loop: branch on `fc.name` and call the helper instead of `rag_search`.

**Guard interaction.** The guard already resolves coreferences; for *"tell me about it"* it produces `query="tell me about the document"`. The synthesizer's tool-choice prompt is updated:
> If the user asks about a document's *shape* (length, sections, what's in it), prefer `fetch_document_outline(document_id=<scope's document_id or the top hit's>)` first. Then optionally one `rag_search` for narrative.

**Estimated LOC.** ~200.

---

### 2.5 Follow-up suggestion chips

The synthesizer already produces `followup` (one). Generate up to 3 instead, in the same JSON turn. UI changes are described in the FE plan.

**Backend changes.**

- Update `synthesizer.py` system prompt to ask for `followup_suggestions: list[str] (max 3)`.
- Update `AgentResponse.followup` from `str | None` to `followups: list[str]` (keep `followup` aliased for one release for backward compat; remove next).
- Update `ChatResponse` schema in `app/routes/chat.py` to include `followups: list[str]`.

**Estimated LOC.** ~40.

---

### 2.6 `fetch_table(document_id, table_id)` tool

**Why.** Tables get chunked in `chunker.py`. A user asking *"show me the full NPT table"* gets one row band. The full table is already serialized in `data/output/.../tables/page_NNN_table_MM.md` (legacy) and Supabase `artifacts/<doc>/tables/...` (new).

**Implementation.**

- New `SupabaseStorage.read_table_markdown(user_id, project_id, document_id, page_num, table_id)` — wraps an existing Storage `download` call.
- New tool declaration with `(document_id, table_id)` params.
- Tool result is plain Markdown injected back to the synthesizer.

**Estimated LOC.** ~100.

---

### 2.7 `compare_documents(doc_ids[], topic)` tool

**Why.** Today multi-doc questions like *"compare NPT in Sajaa 6 vs Sajaa 9"* either burn 2 of 3 rag_search calls on broad fan-outs, or single-search misses one doc.

**Implementation.**

- Behind the scenes: `asyncio.gather` of `rag_search(topic, document_id=d)` for each `d` in `doc_ids`, with a per-doc cap of 5 sources. Returns a structured result tagged by doc.
- Counts as one tool call against the 3-call budget regardless of how many docs.

**Estimated LOC.** ~120.

---

### 2.8 Source verifier — post-pass

**Goal.** Catch `[Sn]` citations the synthesizer dropped on a sentence the source doesn't actually support.

**Implementation.**

- After the synthesizer returns, run a Flash-Lite (cheaper model) pass with the answer + sources and ask: *"For each citation `[Sn]` in the answer, output `(sentence, source_id, supports_yes_no)`."*
- Strike through (HTML `<s>...</s>`) sentences with `supports_yes_no=no`. Or, simpler: mark the citation `[Sn]` with `[Sn?]` so the FE renders it muted.
- Skips when the answer has no citations.

This restores the previous claim validator we removed, but **after** the answer exists — so it doesn't gate response or block streaming.

**Estimated LOC.** ~150.

---

### 2.9 Slash commands

- Parse in `app/routes/chat.py` *before* invoking the agent:
  - `/clear` — `await delete_session(session_id)`; reply `Conversation cleared.`
  - `/forget` — soft-delete all `user_memories` for caller; reply `OK, I've forgotten what I knew.`
  - `/scope project <id>` — return `400` with FE-readable `meta.command="scope"` so the FE can adjust state (no server-side scope state — it's per request).
  - `/help` — return a static help message.
- Zero LLM calls.

**Estimated LOC.** ~80.

---

## 3. Cross-cutting concerns

### 3.1 Token budgeting

- Guard prompt: `system + card(~1k) + history(~600) + memories(~300) + current_subject(~50) + user_msg`. Cap target: 4k input, 1k output.
- Synth prompt per tool turn: `system + initial(~300) + accumulated tool results(≤8k)`. Cap target: 12k input.
- Add a single `app/utils/token_budget.py` helper that truncates oldest history when projected input exceeds `chat_input_token_cap = 16000` (new setting in `app/config.py` — but only when item 1 lands).

### 3.2 Backwards compatibility

Each item is shippable independently. The non-streaming `/chat` endpoint remains the canonical path; `/chat/stream` is additive. Memory items only **add** to `meta`, never break existing fields. Tools are model-internal; they don't change the response contract.

### 3.3 Observability

- Every new path adds one structured log: `mem_capture_done`, `mem_recall_top=N`, `tool_outline_called`, `tool_table_called`, `tool_compare_called`, `verifier_dropped_n`.
- Token metrics: keep the existing `meta.guard_input_tokens / guard_output_tokens` shape; add `meta.synth_input_tokens / synth_output_tokens / verifier_input_tokens / verifier_output_tokens`.

### 3.4 RBAC + tenant isolation

- All new endpoints (`/memories`, `/chat/stream`) inherit `Depends(current_user)`. Owner-only by default; no `?scope=all` for memories.
- All new DB queries scope by `user_id`. Postgres RLS already enforces this on `user_memories` if we mirror the policy used on `documents`.

### 3.5 Tests

For each item:
- Unit test in `tests/test_<item>.py` covering happy path + one edge case.
- New `app_smoke` step that exercises the public behaviour end-to-end.
- No test that hits the live Gemini API in CI; mock `genai.Client` via the existing pattern in `tests/test_agents.py`.

---

## 4. What we are NOT doing in this plan

- New auth modes (passwordless, magic links, SSO).
- Web browsing / external HTTP from the synthesizer.
- Code interpreter, image generation, plugin store.
- Voice I/O.
- Per-document fine-tuning.
- A production `redis-cluster` swap. Single Redis is fine until we are multi-host.
- Any change to the Qdrant schema. All new search behaviour reuses existing collections + payload fields.

---

## 5. File tree of changes (cumulative across all items)

```
app/
  agents/
    rag/
      agent.py            (M — sessions, memory enqueue, streaming entry point)
      guard.py            (M — current_subject + memories block in prompt)
      retriever.py        (unchanged after item 0)
      sessions.py         (NEW — Redis-backed STM)
      synthesizer.py      (M — extra tools, streaming generator, verifier hook)
      types.py            (M — ChatSession.summary + current_subject; followups list)
  db/
    models.py             (M — UserMemory)
    queries/
      context_card.py     (M — memories block)
      document_outline.py (NEW)
      memories.py         (NEW)
  routes/
    chat.py               (M — slash commands, /chat/stream, response shape)
    memories.py           (NEW — list/delete/forget-all)
  storage/
    supabase_storage.py   (M — read_table_markdown helper)
  utils/
    token_budget.py       (NEW)
  workers/
    tasks/
      memory.py           (NEW — extract_user_memory)
supabase/
  migrations/
    <ts>_add_user_memories.sql  (NEW)
tests/
  test_streaming.py       (NEW)
  test_memory_capture.py  (NEW)
  test_outline_tool.py    (NEW)
```

---

## 6. Sequencing — concrete sprint breakdown

**Sprint 1 (foundational).** Items 1 + 2. Streaming gets us the felt-quality jump; STM hardening is a prerequisite for LTM.

**Sprint 2 (memory).** Item 3. Migration → capture → recall → privacy endpoints.

**Sprint 3 (tools).** Items 4 + 5. Outline tool removes the most common broken-feel UX; chips are a free win.

**Sprint 4 (depth).** Items 6 + 7. Table and compare close out the long-tail of "I asked something multi-faceted and got half".

**Sprint 5 (polish).** Items 8 + 9. Verifier raises trust; slash-commands give power users levers.

Each sprint ships independently, behind the relevant feature flag (only one new flag per sprint, named after the item — keeps the guardrail).

---

## 7. Open questions to resolve before each sprint

1. **Streaming.** Do we want SSE or WebSocket? SSE is simpler, one-way, plays nice with our reverse proxy. Recommendation: SSE.
2. **Memory.** Confidence-floor for surfacing memories — start at 0.5 or 0.6? 0.5 favors recall, 0.6 favors precision. Recommendation: 0.5; tune after observing capture-rate.
3. **Outline.** Do we re-derive headings for old documents? Recommendation: lazy — derive on first call, cache to `artifacts/<doc>/headings.json`.
4. **Verifier.** Is a Flash-Lite call per answer too expensive for high-volume users? Recommendation: ship behind a per-user opt-out; default ON.
