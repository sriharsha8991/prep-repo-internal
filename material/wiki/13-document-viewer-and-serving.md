# 13 — Document Viewer & Serving

> _Added 2026-06-07. Owns: how ingested documents are served to the reader
> UI — the endpoints + the lazy/virtualized loading contract for large
> (1000+ page) documents._

## Problem this solves

Naively loading a document — pulling the whole PDF as one `ArrayBuffer`, or the
entire `full_document.md`, or all page images — makes the viewer hang on large
docs (the browser mounts thousands of DOM nodes/canvases). The backend serves
**per-page, range-capable, cacheable** primitives so the FE can lazy-load only
what's in view.

## Serving endpoints  ·  [app/routes/documents.py](../app/routes/documents.py)

| Endpoint | Returns | Notes |
|---|---|---|
| `GET /documents/{id}/manifest` | `{ page_count, ingest_status, pages_done, headings[] }` | Lightweight. Powers virtualization + table-of-contents. **No page content.** |
| `GET /documents/{id}/pages/{n}/markdown` | one page's markdown (`text/plain`) | Lazy per-page text. Falls back to `page_NNN.state.json` if no `.md` sidecar. `Cache-Control: max-age=86400`. |
| `GET /documents/{id}/pages/{n}/image?w=&fmt=` | rendered PNG/WebP **bytes** | `w ∈ {160,320,800,1600}` downscale; WebP via `?fmt=webp` or `Accept`. Immutable-cached. **200 OK with bytes — not a 302.** |
| `GET /documents/{id}/pages/{n}/image/crop?ymin&xmin&ymax&xmax` | bbox crop (0..1000 coords) | Evidence cards. Immutable-cached. |
| `GET /documents/{id}/file` | source PDF (`FileResponse`) | `Accept-Ranges: bytes` + `206` so pdf.js fetches only viewed pages' byte ranges. |
| `GET /documents/{id}/full` | entire `full_document.md` | Legacy whole-doc blob. Prefer per-page markdown for the reader. |

All text/JSON responses are gzip-compressed (`GZipMiddleware`, `minimum_size=500`,
[app/main.py](../app/main.py)).

## Frontend contract (lazy/virtualized reader)

1. **Fetch `/manifest` first** → render `page_count` fixed-height placeholders +
   the heading TOC (jump-to-page). All pages `1..page_count` have rendered PNGs.
2. **Virtualize** the page list (windowing): mount real content only for pages
   in/near the viewport; unmount when far away.
3. **Lazy-load per page** via `IntersectionObserver`:
   - image reader (recommended): `pages/{n}/image?w=320` for the rail,
     `?w=1600` for the active page — rendering already done at ingest, served
     from the immutable cache;
   - text reader: `pages/{n}/markdown`;
   - pdf.js reader: pass the `/file` **URL** (not a downloaded buffer) with
     `disableAutoFetch:true, disableStream:false` so it range-fetches.
4. **AbortController** to cancel in-flight fetches on fast scroll; an **LRU**
   cap on rendered pages bounds client memory.

## Operational notes
- The manifest reads `full_document.md` server-side to parse headings but
  returns only the small outline — cache later if it shows up hot.
- Per-page `.md` sidecars are written in Phase 3 postprocessing
  ([postprocessor.py](../app/pipeline/postprocessor.py)); the state-JSON
  fallback covers pages written before sidecars existed.

## Things this page does not own
- Where the bytes live → [05-storage-and-vectors.md](05-storage-and-vectors.md).
- How they're produced → [03-ingestion-pipeline.md](03-ingestion-pipeline.md).
- Full endpoint contracts → [07-api-reference.md](07-api-reference.md).
