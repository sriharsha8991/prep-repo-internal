# 05 — Storage & Vectors

> _Changed 2026-06-07: rewritten for the Supabase→local migration. Owns:
> where bytes physically live (local filesystem) and how vectors are
> organised in Qdrant._

## Local filesystem storage  ·  [app/storage/local_storage.py](../app/storage/local_storage.py)

Supabase Storage is gone. Artifacts live on a local volume under
`storage_root` (`./data/storage`, mounted into the app + worker containers).
Path scheme: `<storage_root>/<bucket>/<user_id>/<project_id>/<document_id>/<file>`

| Bucket (sub-dir) | Holds | Files |
|---|---|---|
| `pdfs` | Original uploaded PDF | `source.pdf` |
| `pages` | Rendered page PNGs (adaptive DPI) | `page_NNN.png` |
| `artifacts` | Markdown / tables / entities / per-page state | `page_NNN.md`, `page_NNN.state.json`, `full_document.md`, `tables/*`, `entities.json` |

There is **no signed-URL layer and no RLS**. Ownership is enforced in
application code: every read/write includes the `user_id`/`project_id`/
`document_id` path prefix, and routes verify ownership via `_resolve_doc`
(super_admin may read any). Image serving returns **raw image bytes**
(`200 OK`), not a 302 redirect — see
[13-document-viewer-and-serving.md](13-document-viewer-and-serving.md).

Key methods: `write_pdf`/`read_pdf`/`file_path_pdf` (range streaming),
`write_page_png`/`read_page_png`, `write_page_markdown`/`read_page_markdown`,
`write_page_state`/`read_page_state`, `read_full_document`, `delete_document`.

## Qdrant collections  ·  [app/vectorstore/qdrant_store.py](../app/vectorstore/qdrant_store.py)

Collections are **org-scoped**. The default (null-org / super_admin) names:

| Default collection | Vector | One point per | Purpose |
|---|---|---|---|
| `wellsynthai_documents` | 768-d (`gemini-embedding-001`) | ~1000-char chunk | Text retrieval |
| `wellsynthai_entities` | 768-d | resolved entity | Entity search (wells, formations, units) |
| `wellsynthai_images` | 768-d | rendered page / image description | Image grounding for chat |

An org gets its own collection set (created lazily on first ingest via
`for_org(organization, role)` → `ensure_collection()`).

> Embeddings are **768-d** (`embedding_dimensions=768`, `embedding_model=gemini-embedding-001`),
> not 1536-d. Update both config and any reset tooling if this changes.

### Payload on every point
```json
{
  "user_id": "uuid", "role": "drilling_engineer | ...",
  "organization": "Acme Drilling", "project_id": "uuid",
  "document_id": "uuid", "page_num": 4,
  "text": "...",        // chunks/entities
  "section": "...",     // chunks
  "table_id": "...",    // table chunks
  "entity_kind": "...", // entities
  "png_path": "..."     // local storage key for the source page image
}
```
`user_id`, `project_id`, `document_id` are indexed payload fields.

### Mandatory filter
Every read filters on `user_id` (plus optional `project_id`/`document_id`/
`page_num`). **No Qdrant query runs without `user_id`** — a missing filter is a
security bug.

## Why Qdrant, not pgvector
- Separate text/image vector spaces; high concurrent insert volume during
  ingest; filter + score hybrid search on the roadmap.

## Resetting Qdrant
[app/tools/qdrant_reset.py](../app/tools/qdrant_reset.py) → `store.reset_all()`
wipes and recreates the collection set (respecting org-scoped naming). Run
**only** if you also re-ingest, or users see "no documents found".

## Things this page does not own
- The pipeline that produces these vectors → [03-ingestion-pipeline.md](03-ingestion-pipeline.md).
- Serving page images/markdown to the viewer → [13-document-viewer-and-serving.md](13-document-viewer-and-serving.md).
- How retrieval consumes them → [04-retrieval-and-chat.md](04-retrieval-and-chat.md).
- Postgres tables → [06-database-schema.md](06-database-schema.md).
