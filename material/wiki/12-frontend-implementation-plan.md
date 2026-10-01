# Wellsynthai — Frontend Implementation Plan: Memory + ChatGPT-feel Roadmap

> **Companion document.** This plan covers only the frontend changes
> required to land the items defined in the backend plan
> (`11-memory-and-chatgpt-roadmap.md`). Read that first — endpoint
> contracts, response shapes, and feature ordering live there.
>
> **Stack assumed.** React (TypeScript) + the existing Wellsynthai
> dashboard. State via component-level hooks + a single chat-feature
> store (Zustand or equivalent — already in use). Network via the
> existing typed API client. No new framework.
>
> ⚠️ **Pre-migration note (2026-06-07).** Auth is **no longer** Supabase —
> `@supabase/supabase-js` was removed. The FE calls the backend's self-hosted
> `/auth/login` + `/auth/refresh` (HS256 JWT) directly and attaches a Bearer
> token. Job status is **polled** (`GET /ingest/{job_id}`), not subscribed via
> Supabase Realtime. The memory/UX plan below otherwise stands.
>
> **Discipline.** Match the backend sprint cadence. Don't pre-build UI
> for items the backend hasn't shipped yet.

---

## 0. Top-level UX shape

Today's chat surface is a transcript list + composer + a "Sources (N)"
disclosure. The end state after these sprints:

```
┌───────────────────────────────────────────────────────────┐
│  Chat                                          ⚙  Memory  │   ← settings entry point
│  session abcd-1234           [All projects ▾]  [⌫]  [↻]  │   ← /clear, regenerate
├───────────────────────────────────────────────────────────┤
│  user: …                                                  │
│                                                           │
│  assistant (streaming): The 9-5/8" casing was set at…▮    │   ← cursor
│    [S1?] muted — verifier flagged                         │   ← optional
│    ▸ Sources (10)                                         │
│    ▸ Pages (2 attached)                                   │
│    suggested follow-ups:                                  │
│    [Why was this depth chosen?] [Compare to Sajaa 9]      │
│    [Show full NPT table]                                  │
├───────────────────────────────────────────────────────────┤
│  composer:  type a message or `/clear`, `/forget`, `/help`│
└───────────────────────────────────────────────────────────┘
```

Plus a small **memories** drawer reachable from `⚙ Memory` showing
"What WellSynth remembers about you" with per-row delete + a
"Forget everything" button.

---

## 1. Recommended frontend implementation order

Mirrors the backend order so each sprint's BE + FE land together.

| # | Sprint | Frontend deliverable |
|---|---|---|
| 1 | Streaming | Switch to SSE; show streaming cursor; "Stop generating" button |
| 2 | STM hardening | (No FE change — purely backend) |
| 3 | Long-term memory | "Memory" drawer + capture toast + privacy controls |
| 4 | Outline tool | (No new UI — synthesizer call is internal) |
| 5 | Follow-up chips | Render up to 3 chips below assistant turn; click sends message |
| 6 | `fetch_table` tool | (No new UI — table renders inline as before) |
| 7 | `compare_documents` | (Optional) tagged source rows by doc name |
| 8 | Source verifier | Render `[Sn?]` muted + tooltip on hover |
| 9 | Slash commands | Composer hint + intercept handlers |

---

## 2. Item-by-item plan

### 2.1 Streaming chat (SSE)

#### Endpoint contract recap

`POST /chat/stream` → `text/event-stream`. Each line is
`data: <json>`. Possible `event` values inside the JSON:

- `{"event":"status","phase":"guard"}`
- `{"event":"status","phase":"rag_search","query":"…"}`
- `{"event":"token","text":"…"}` (concatenate in order)
- `{"event":"done", "sources":[...], "attached_pages":[...], "followups":[...], "meta":{...}}`

If the connection drops mid-stream, the FE displays the partial text
plus a small `↻ retry` button.

#### Files

- `frontend/src/lib/api/chatStream.ts` (NEW). Exposes
  ```ts
  export function chatStream(
    body: ChatRequest,
    handlers: {
      onStatus?(phase: 'guard'|'rag_search', query?: string): void;
      onToken?(t: string): void;
      onDone?(payload: ChatDonePayload): void;
      onError?(e: Error): void;
    },
    signal: AbortSignal,
  ): Promise<void>;
  ```
  Internally uses `fetch()` with `headers: { Authorization: Bearer ${session.access_token} }` and reads `response.body!.getReader()` line-by-line. (Native `EventSource` won't carry custom Authorization headers — that's why we use `fetch` with manual SSE parsing.)
- `frontend/src/features/chat/useStreamingChat.ts` (NEW). A hook that:
  1. Pushes the user message into the transcript.
  2. Pushes a placeholder assistant message with `streaming: true`.
  3. Calls `chatStream`, appending tokens to the placeholder, and finalising the message on `done`.
  4. Exposes `stop()` which calls `abort()`.
- `frontend/src/features/chat/ChatView.tsx` (M). Add:
  - A `▮` blinking caret on the streaming message.
  - A floating "Stop generating" pill while `streaming === true`.
  - A small status line above the bubble: `"Searching documents…"` when the latest status is `rag_search`, `"Thinking…"` when it's `guard`.
- `frontend/src/features/chat/state.ts` (M). Message type gains
  `streaming?: boolean`, `phase?: 'guard'|'rag_search'`.

#### Edge cases

- **Network drop:** show partial + retry; do not auto-resume (no resume token).
- **User sends new message while streaming:** call `stop()` on the previous, then begin the new.
- **"Stop generating":** call `abort()`, mark `streaming=false`, keep the partial answer in the transcript.
- **Token rate spikes:** `requestAnimationFrame`-coalesce token appends to avoid layout thrash.
- **Empty `done` payload:** treat as if non-streaming `/chat` returned no answer; show the existing "no answer" fallback string.

#### Estimated LOC

~350 frontend.

---

### 2.2 STM hardening

**FE work: none.** Sessions are still keyed by `session_id`; the FE keeps doing what it does. Acknowledge by adding a single regression test: after a backend restart, the same `session_id` should still produce coreferent answers.

---

### 2.3 Long-term memory — UI

#### A. "Memory" drawer

Reachable from a **⚙ Memory** entry in the chat header.

Components (all new):

- `frontend/src/features/memory/MemoryDrawer.tsx`
- `frontend/src/features/memory/MemoryRow.tsx`
- `frontend/src/lib/api/memories.ts` — typed wrappers for:
  - `GET    /memories?scope=`
  - `DELETE /memories/{scope}/{key}`
  - `POST   /memories/forget-all`

UX:

```
┌──────────────────────────────────────────────────────────┐
│  What WellSynth remembers about you                  ✕   │
├──────────────────────────────────────────────────────────┤
│  Preferences                                             │
│   • Always uses metric units            [delete]         │
│   • Prefers concise replies              [delete]        │
│  Facts                                                   │
│   • Drilling engineer at PetroX          [delete]        │
│  Project pins                                            │
│   • Default project: Sajaa               [delete]        │
│  Style                                                   │
│   • Tone: neutral, length: short                         │
│                                                          │
│  [Forget everything]                                     │
└──────────────────────────────────────────────────────────┘
```

- Group rows by `scope`. Hide the `dismissed` scope from this view.
- "Forget everything" requires a typed-confirmation modal ("type FORGET to confirm").
- Shows `Last updated <relative>` per row (mute small text).

#### B. Capture toast

When the chat response's `meta.memory_captured: true` (a flag the BE
adds when the post-turn Celery task fires), show a small unobtrusive
toast: *"Saved a note about your preferences. ⚙ Manage."* The link
opens the drawer.

This makes capture visible and consensual without being noisy. Toast
auto-dismisses after 4s.

#### C. Permission posture

Memory is on-by-default per backend choice. Surface this once on first
visit via a small banner in the chat header: *"WellSynth remembers
preferences across sessions to give better answers. You can manage or
turn this off in ⚙ Memory."* Dismiss persisted in localStorage.

#### D. Files

```
frontend/src/features/memory/
  MemoryDrawer.tsx          (NEW)
  MemoryRow.tsx             (NEW)
  ForgetAllConfirm.tsx      (NEW)
  useMemories.ts            (NEW — fetch, delete, invalidate)
frontend/src/lib/api/
  memories.ts               (NEW)
frontend/src/features/chat/
  ChatView.tsx              (M — toast on memory_captured)
  ChatHeader.tsx            (M — ⚙ Memory entry + first-visit banner)
```

#### Estimated LOC

~600 frontend.

---

### 2.4 `fetch_document_outline` tool

**FE work: none required.** The outline call is internal to the
synthesizer. Optional polish: when `meta.tools_used` includes
`fetch_document_outline`, render a small subtle badge under the answer
("Outline used"). Keep it for debugging power users — hide behind the
existing dev flag.

---

### 2.5 Follow-up suggestion chips

#### Contract change

Response now has `followups: string[]` (max 3) instead of `followup: string | null`. Both FE clients (chat + dev tools) need to read the new field and fall back to `[followup]` for one release.

#### UI

- Below the assistant message, show up to 3 chips. Compact pill style.
- Click → fills the composer with the suggestion **and** auto-sends.
- Hidden when the message is still streaming.
- If all chips are present in `last_user_message_chips_clicked`, hide them (don't re-suggest the same prompt the user already asked).

#### Files

```
frontend/src/features/chat/FollowupChips.tsx   (NEW)
frontend/src/features/chat/MessageBubble.tsx   (M — render chips)
frontend/src/features/chat/state.ts            (M — chips_clicked: Set<string>)
```

#### Estimated LOC

~120.

---

### 2.6 `fetch_table` tool

**FE work: none.** Table comes back as Markdown, rendered by the existing markdown renderer. Optional polish: if a single fenced markdown table dominates the answer, lift it out into the existing **Tables** carousel.

---

### 2.7 `compare_documents` tool

**Optional polish.** When `meta.tools_used` includes
`compare_documents`, the **Sources** disclosure groups source rows by
`doc_name`:

```
▾ Sources (10)
  Sajaa 6 EOWR (5)
    [S1] p14 — NPT for Sajaa 6 was 9.8% of total time…
    …
  Sajaa 9 EOWR (5)
    [S6] p11 — NPT for Sajaa 9 was 11.4%…
    …
```

#### Files

```
frontend/src/features/chat/SourcesPanel.tsx    (M — group by doc_name when tool='compare')
```

#### Estimated LOC

~80.

---

### 2.8 Source verifier — muted citations

#### Contract

The answer text contains either `[S1]` (verified) or `[S1?]` (verifier marked unsupported). The FE existing markdown post-processor already linkifies `[Sn]` to a citation chip — extend it to recognise `[Sn?]` and render it greyed out + a tooltip:
*"This claim wasn't fully supported by the cited source."*

#### Files

```
frontend/src/features/chat/markdown/citationsPlugin.ts  (M)
frontend/src/features/chat/CitationChip.tsx             (M — supported?: boolean)
```

#### Estimated LOC

~60.

---

### 2.9 Slash commands

#### Recognised commands

Parsed client-side in the composer when the message starts with `/`:

- `/clear` → fully local: clear transcript, mint a new `session_id`. Optionally fire a backend hint so the BE drops Redis state.
- `/forget` → POST `/memories/forget-all`; show toast.
- `/scope project <name>` → resolve `<name>` → `project_id` from the loaded project list; update the project filter dropdown; *don't* send a message.
- `/scope document <id-prefix>` → same shape.
- `/help` → show a help dialog listing all commands.

If the slash command isn't recognised, treat as a normal message.

#### UX

- As the user types `/`, a small autocomplete popover lists the available commands with one-line descriptions.
- Tab autocompletes; Enter executes.
- Composer placeholder shows: `type a message or /help`.

#### Files

```
frontend/src/features/chat/Composer.tsx           (M — / detection, popover)
frontend/src/features/chat/SlashCommands.ts       (NEW — registry + parser)
frontend/src/features/chat/HelpDialog.tsx         (NEW)
```

#### Estimated LOC

~250.

---

## 3. Cross-cutting frontend concerns

### 3.1 API client typing

Add types for all new shapes in `frontend/src/lib/api/types.ts`:

```ts
export interface ChatDonePayload {
  sources: Source[];
  attached_pages: AttachedPage[];
  followups: string[];
  meta: ChatMeta;
}

export interface ChatMeta {
  path: 'prefilter' | 'guard_reply' | 'guard_clarify' | 'guard_then_synth';
  rag_invoked: boolean;
  guard_input_tokens: number;
  guard_output_tokens: number;
  rag_calls?: number;
  tools_used?: string[];          // NEW
  memory_captured?: boolean;      // NEW
  verifier_dropped?: number;      // NEW
}

export interface MemoryItem {
  scope: 'preference' | 'fact' | 'project_pin' | 'style';
  key: string;
  value: unknown;
  confidence: number;
  updated_at: string;
}
```

### 3.2 Auth token plumbing

`/chat/stream` uses bearer auth via headers. Memory endpoints reuse the
existing `apiClient` wrapper which already injects
`Authorization: Bearer ${session.access_token}`. **No** cookies, **no**
query-string tokens — they would leak into log files.

### 3.3 Optimistic UI rules

- User message: optimistically rendered the moment send is called.
- Assistant message: skeleton bubble appears as soon as SSE opens (`status: guard`). No optimistic *content*.
- Memory drawer: optimistic delete with rollback on 4xx.

### 3.4 Accessibility

- "Stop generating" pill is a real `<button>` with aria-label.
- Streaming caret is announced once via `aria-live="polite"` (`"Assistant is responding…"`), then silenced.
- Memory drawer is a focus-trapped dialog.
- Slash-command popover uses `role="listbox"` with arrow-key navigation.
- All new icons have text alternatives.

### 3.5 Error handling

- Stream open fails → show the existing inline error banner; the user can retry.
- Memory fetch fails → drawer shows a retry CTA; never silently empty.
- Slash-command resolution fails (e.g. unknown project name) → toast: *"No project named X."*
- Verifier-tagged citations failing to load metadata → render plain `[Sn]` (degrade gracefully).

### 3.6 Telemetry

Emit FE analytics events (using whatever pipeline is already wired —
do not introduce a new SDK):

- `chat.stream.opened`, `chat.stream.completed`, `chat.stream.aborted`
- `memory.drawer.opened`, `memory.row.deleted`, `memory.forget_all`
- `chat.slash.executed` with `{ command }`
- `chat.followup.clicked`

### 3.7 Regression coverage

Per item, add at least one Cypress (or Playwright — match existing) test:

- `chat-streaming.spec.ts` — sends a question, verifies tokens stream, verifies "Stop generating" works, verifies final source list.
- `memory-drawer.spec.ts` — opens drawer, deletes a row, verifies row gone.
- `followup-chips.spec.ts` — clicks a chip, verifies new user message in transcript.
- `slash-commands.spec.ts` — types `/clear`, verifies new session id and empty transcript.

### 3.8 Feature flags

Mirror the backend's per-sprint flag. When the backend feature flag is
off, the FE component is hidden:

- `ff.streaming` → `/chat/stream` is enabled.
- `ff.memory` → "⚙ Memory" entry is visible; capture toast fires.
- `ff.outline_tool`, `ff.compare_tool`, `ff.verifier`, `ff.slash_commands`
  → corresponding UI affordances visible.

Flags arrive from `/auth/me` (already returns user role + org); add a
nested `ff` object on the BE response.

---

## 4. Component-level diff summary

```
frontend/src/
  features/
    chat/
      ChatView.tsx          (M — streaming, status line, memory toast)
      ChatHeader.tsx        (M — ⚙ Memory, first-visit banner)
      Composer.tsx          (M — slash commands)
      MessageBubble.tsx     (M — streaming caret, chips, citation chips)
      FollowupChips.tsx     (NEW)
      SourcesPanel.tsx      (M — grouped-by-doc when compare)
      SlashCommands.ts      (NEW)
      HelpDialog.tsx        (NEW)
      markdown/
        citationsPlugin.ts  (M — recognise [Sn?])
      state.ts              (M — streaming, chips_clicked, ff)
      useStreamingChat.ts   (NEW)
    memory/
      MemoryDrawer.tsx      (NEW)
      MemoryRow.tsx         (NEW)
      ForgetAllConfirm.tsx  (NEW)
      useMemories.ts        (NEW)
  lib/
    api/
      chatStream.ts         (NEW)
      memories.ts           (NEW)
      types.ts              (M — new types above)
```

---

## 5. Sprint breakdown (FE-side)

**Sprint 1.** `chatStream.ts` + `useStreamingChat.ts` + ChatView streaming polish + "Stop generating" button + cypress test. Ship behind `ff.streaming`.

**Sprint 2.** Nothing — STM is BE-only.

**Sprint 3.** Memory drawer + API client + capture toast + first-visit banner + privacy modal + cypress.

**Sprint 4.** Optional dev-flag badge for outline-tool usage. No user-visible UI.

**Sprint 5.** Follow-up chips component + state changes + cypress.

**Sprint 6.** Optional source-grouping when `compare_documents` was used.

**Sprint 7.** Citation chip rendering for `[Sn?]` + tooltip.

**Sprint 8.** Slash-command parser + popover + help dialog + cypress.

---

## 6. Open questions for the FE team

1. **Streaming markdown rendering.** Do we render markdown incrementally token-by-token (mark may produce flicker on tables) or buffer until a paragraph break? Recommendation: buffer until a newline, render in chunks; render the full markdown on `done`.
2. **Memory drawer placement.** Header icon vs left-rail entry? Recommendation: header icon, right side, beside the project filter.
3. **Slash-command `/scope`.** Should this auto-send a "scoped" message or just adjust the dropdown silently? Recommendation: silent dropdown adjustment, with a toast confirming.
4. **Follow-up chip styling.** Pill vs tag vs inline link. Recommendation: pill, primary-tinted on hover.
5. **Citation chip for `[Sn?]`.** Strikethrough vs muted vs orange dot. Recommendation: muted with a small `?` superscript and tooltip on hover.

---

## 7. Out of scope for this plan

- Voice input/output.
- Server-pushed notifications outside the SSE chat stream.
- Mobile-only redesign of chat surface.
- A new component library (we keep the existing primitives).
- Dark-mode-only changes.
- Internationalisation. (English only for now; new strings live in the same i18n file regardless, ready for later translation.)
