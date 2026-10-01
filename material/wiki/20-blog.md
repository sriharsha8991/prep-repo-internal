# 20 — Marketing Blog

> _Added 2026-07-29. Owns: the public blog surface, the super_admin authoring
> API, and the blog media pipeline (upload → storage → anonymous serving).
> For the FE-authored wire contract see `docs/blog_api_contract.md` (local
> only — `/docs` is gitignored). For the tables' place in the wider schema see
> [Database Schema](06-database-schema.md); for the storage buckets this adds
> to, [Storage & Vectors](05-storage-and-vectors.md)._

A super_admin writes a post in a markdown editor inside the app and publishes
it. It appears immediately on the public marketing blog — **no rebuild, no
deploy**. Posts support markdown, uploaded images, YouTube/Vimeo embeds and
links.

## Three inversions — read this first

Almost every instinct the rest of this backend has trained into you is **wrong**
here. The blog deliberately inverts three invariants that are load-bearing
everywhere else:

| Everywhere else | The blog |
|---|---|
| Every table is tenant-scoped (`organization_id`) and every read is scoped by caller | **No tenant.** One global marketing surface. `blog_posts` has no `organization_id` and that absence is the design, not an oversight. |
| Every route requires a JWT; `request.state.user` is always populated | **`GET /blog/*` is anonymous.** The middleware never populates `request.state.user` there — reading it is a crash, not a 401. |
| Every image is JWT-gated and rendered through `AuthImage` (which fetches bytes with a bearer token) | **Blog images are public bytes.** The FE renders them with a plain `<img>`. Add an auth check to the media route and every image in every post breaks for the public. |

If you are changing something here and it feels like it violates a rule from
[Auth & RBAC](02-auth-and-rbac.md), that is expected — check this table before
"fixing" it.

## Two surfaces, two postures

| Surface | Audience | Auth | Router |
|---|---|---|---|
| `GET /blog/*` | Anonymous visitors on the marketing site | **None.** No `Authorization` header is sent. | [app/routes/blog.py](../app/routes/blog.py) |
| `/admin/blog/*` | The author | **Bearer JWT + `super_admin`** | [app/routes/admin_blog.py](../app/routes/admin_blog.py) |

They are **separate modules on purpose.** The security postures are opposite,
and one file holding both is how that boundary gets blurred by a later edit.

### What actually makes the public routes public

```python
# app/auth/middleware.py
ANON_PREFIXES: tuple[str, ...] = ("/static/", "/blog/")
```

**That tuple, not a decorator, is the mechanism.** Consequences:

- Anything mounted under `/blog/` is silently public. This is why authoring
  lives at `/admin/blog/*` and **must stay there**.
- `OPTIONS` short-circuits before the token check (`AuthMiddleware.dispatch`),
  so anonymous CORS preflight never 401s.

## Request path

```
  ANONYMOUS READER                          SUPER_ADMIN AUTHOR
  (www.wellsynth.ai)                        (app.wellsynth.ai)
        │                                          │
        │  no Authorization header                 │  Bearer JWT
        ▼                                          ▼
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │ AuthMiddleware           │            │ AuthMiddleware           │
  │  _is_anonymous("/blog/") │            │  verify_token → profile  │
  │  → bypass entirely       │            │  → request.state.user    │
  └───────────┬──────────────┘            └───────────┬──────────────┘
              │                                       │ require_role("super_admin")
              ▼                                       ▼
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │ routes/blog.py           │            │ routes/admin_blog.py     │
  │  _rate_limit (per IP)    │            │  validate + normalize    │
  │  published-only reads    │            │  slug freeze enforcement │
  └───────────┬──────────────┘            └───────────┬──────────────┘
              │                                       │
              └──────────────┬────────────────────────┘
                             ▼
                 ┌──────────────────────────┐
                 │ db/queries/blog.py       │
                 │  _published_only() gate  │  ← the single leak-prevention point
                 │  slugify / reading_min   │
                 └───────────┬──────────────┘
                             ▼
              Postgres: blog_posts, blog_media
              LocalStorage: <storage_root>/blog/<media_id>/image.<ext>
```

**No Celery, no credits, no Gemini.** Authoring costs no model calls, so nothing
on either router takes a credit hold — this is the one route group in the
backend that is exempt from the reserve/settle discipline in
[Credits & Billing](18-credits-and-billing-explained.md), because there is
nothing to meter.

## Data model  ·  [migrations/028_blog.sql](../migrations/028_blog.sql) · [Alembic 017](../alembic/versions/20260729_017_blog.py)

### `blog_posts`

| Column | Type | Notes |
|---|---|---|
| `id` | UUID PK | |
| `slug` | TEXT | **Unique, frozen once published.** The public URL key. |
| `title` | TEXT | |
| `excerpt` | TEXT | |
| `body_markdown` | TEXT | **Stored verbatim** — see below |
| `cover_image_url` | TEXT | Publicly readable URL |
| `tags` | `TEXT[]` | Free text; GIN-indexed |
| `author_name` / `author_title` | TEXT | Display byline, not a FK |
| `status` | TEXT | `draft` \| `published` |
| `published_at` | TIMESTAMPTZ | NULL while draft. Future value = scheduled. |
| `seo_title` / `seo_description` / `canonical_url` | TEXT | |
| `reading_minutes` | INTEGER | Computed server-side on every write |
| `author_user_id` | UUID | Who wrote it (nullable, not enforced) |
| `created_at` / `updated_at` | TIMESTAMPTZ | |

Indexes and why each exists:

| Index | Why |
|---|---|
| `blog_posts_slug_uq` (UNIQUE) | **This is the real uniqueness guarantee.** The route maps its `IntegrityError` to 409, which stays correct under concurrent writes in a way a pre-flight `SELECT` would not. |
| `blog_posts_published_idx` (partial, `WHERE status='published'`) | The public index feed, newest first. |
| `blog_posts_updated_idx` | Admin list ordering (`updated_at DESC`). |
| `blog_posts_tags_gin` | Exact tag filter. The query uses `tags @> ARRAY[?]` (`.contains()`) — `.any()` would emit `= ANY(tags)` and seq-scan instead. |

### `blog_media`

`id`, `storage_key`, `content_type`, `bytes`, `width`, `height`,
`original_filename`, `uploaded_by`, `created_at`.

The row is the index; the bytes live on disk. `storage_key` points into the
`blog` bucket.

### `body_markdown` is stored verbatim

No server-side markdown→HTML, no rewriting, **no sanitising that alters the
text.** The FE owns rendering (`react-markdown` + `rehype-raw → rehype-sanitize
→ rehype-katex`) and sanitises at render time.

Videos are stored as a **bare URL on its own line**, not as an `<iframe>` — the
renderer detects the line and builds the player itself, so the sanitiser never
has to allow iframes.

## The slug lifecycle

```
   create ──────────────► draft ──── publish ────► published
   slug = slugify(title)    │  ▲                      │
   or author override       │  └──── unpublish ───────┘
                            │        (keeps slug AND published_at)
                            │
                     slug editable          slug FROZEN
                                            → 409 blog_slug_immutable
```

`slugify()` lives in [app/db/queries/blog.py](../app/db/queries/blog.py) and
**must stay identical to the FE's** `slugify` — NFKD-fold to ASCII, lowercase,
non-alphanumeric runs → `-`, strip edges.

Why the freeze: once published, the URL is in someone's feed, someone's
bookmark, and Google's index. Renaming would 404 all of them. The editor
disables the field for published posts and never sends it; a request that
reaches the server is a bug or a hand-rolled call, so the server **refuses
rather than silently ignoring**.

Unusable input (`"!!!"` → `""`) is rejected with `blog_slug_empty` rather than
writing a post that has no reachable URL.

### `reading_minutes`

`round(words / 225)`, minimum 1, recomputed on **every** write that touches the
body — so the stored value can never drift from the text it describes. The FE
has an identical local fallback; **keep the two formulas in lockstep** or the
card and the post disagree by a minute.

## Images — the whole pipeline

This is the part most likely to be got wrong, because it inverts the product's
normal posture completely.

### Why blog images are not like any other image here

Every other image in the product is served JWT-gated and rendered through
`AuthImage`, which fetches the bytes with a bearer token (see
[Document Viewer & Serving](13-document-viewer-and-serving.md)). **Blog readers
have no JWT.** The FE renders blog images with a plain `<img>` tag. If the URL
requires a token, every image in every post renders broken for the public.

### Upload → serve

```
  EDITOR                     POST /admin/blog/media    (multipart, field "file")
    │                        super_admin only
    ▼
  ┌────────────────────────────────────────────────────────────┐
  │ 1. VALIDATE  content_type ∈ {png, jpeg, webp, gif, svg+xml}│ → 400 unsupported_media_type
  │              len(bytes) ≤ BLOG_MEDIA_MAX_BYTES (5 MB)      │ → 400 media_too_large
  │ 2. PROBE     Pillow → (width, height); SVG → (None, None)  │   never raises
  │ 3. WRITE     LocalStorage.write_blog_media(media_id, ...)  │
  │                <storage_root>/blog/<media_id>/image.<ext>  │
  │ 4. RECORD    blog_media row, id == the id used in the path │
  │ 5. RETURN    { url, content_type, bytes, width?, height? } │
  │                url = f"{BLOG_MEDIA_BASE_URL}/blog/media/{id}"
  └────────────────────────────────────────────────────────────┘
    │
    ▼  editor inserts the ABSOLUTE url into body_markdown
  ![alt](https://api.wellsynth.ai/blog/media/9c3fac3e-…)
    │
    ▼  later, for an anonymous reader
  GET /blog/media/{id}   → 200 image bytes, no auth
```

The media id is generated **before** the write and passed into both the storage
key and the row, so the path on disk and the DB row always point at each other.

### Storage layout

Adds one bucket to the scheme in [Storage & Vectors](05-storage-and-vectors.md):

```
<storage_root>/blog/<media_id>/image.<ext>
```

Note it is **not** under the `<user_id>/<project_id>/<document_id>` tenant path
every other bucket uses. These bytes are served to anonymous readers, so the
path deliberately carries no tenant identity to leak.

### The absolute-URL trap (the single most important thing on this page)

`BlogMediaUpload.url` is **absolute**, and it is built from
`settings.blog_media_base_url` — **not** from `request.base_url`.

Two independent reasons, both of which have already bitten:

1. **Relative would resolve to the wrong host.** The blog is served from the
   SWA origin (`www.wellsynth.ai`); the API is `api.wellsynth.ai`. A root-
   relative `/blog/media/x` resolves against the frontend and 404s.
2. **The URL is permanent.** It is written into `body_markdown` and stays
   there. Behind nginx, `request.base_url` reports `http://` unless forwarded
   headers are explicitly trusted — so one bad deploy would bake **mixed
   content into published posts forever**, with no way to correct it short of
   rewriting every row.

There is no rebase for old posts. Changing `BLOG_MEDIA_BASE_URL` later does
**not** fix posts already published against the old value.

### Per-environment configuration

Set in each compose file's `app` service:

| Environment | Compose file | `BLOG_MEDIA_BASE_URL` |
|---|---|---|
| Local | [docker-compose.local.yml](../docker-compose.local.yml) | `http://localhost:8080` |
| Dev | [docker-compose.dev.yml](../docker-compose.dev.yml) | `https://api-dev.wellsynth.ai` |
| Prod | [docker-compose.yml](../docker-compose.yml) | `https://api.wellsynth.ai` |

The config default is the **prod** origin, so an environment that forgets to set
it fails toward prod URLs — wrong, but loudly wrong on the first upload.

**Verify before uploading**, not after:

```bash
curl http://localhost:8080/blog/config
# {"media_base_url":"http://localhost:8080","environment":"local"}
```

If that shows the wrong origin, the container is running stale config —
restart it, then re-upload. Images already uploaded keep the old URL.

### Serving + hardening

`GET /blog/media/{id}` returns raw bytes (`200`, not a redirect) with:

| Header | Why |
|---|---|
| `Cache-Control: public, max-age=31536000, immutable` | The id is a uuid and its bytes never change — a new upload is a new id. |
| `X-Content-Type-Options: nosniff` | No MIME sniffing. |
| `Content-Security-Policy: default-src 'none'; sandbox` | **SVG is in the allowlist** (the editor permits it), and an SVG can carry `<script>` that would execute on the API's own origin. Upload is super_admin-only so this is defence in depth, not a live hole — but it costs nothing. |
| `Content-Disposition: inline` | Rendered, not downloaded. |

### Known gaps

- **Single-host disk.** There is no blob store or CDN in this deployment;
  `LocalStorage` is the API host's filesystem. Blog images therefore ride the
  API's uptime and disk. Moving to Azure Blob + CDN later means changing
  `_media_url()` and the storage writer — the DB row and the markdown URLs of
  *existing* posts would still need rewriting.
- **No dedup.** Uploading the same file twice stores it twice.
- **No orphan GC.** Deleting a post does not delete media it referenced (the
  reference lives inside free-text markdown, so there is nothing reliable to
  cascade on).

## API contract

Error shape matches the rest of the admin console: FastAPI
`{"detail": {"error_code": "...", ...}}`.

### Public — no auth

| Endpoint | Returns |
|---|---|
| `GET /blog/posts?tag=&q=&limit=24&offset=0` | `BlogListResponse`. **Published only** (`status='published' AND published_at <= now()`), `published_at DESC`. No `body_markdown`. |
| `GET /blog/posts/{slug}` | `BlogPost` · `404` if unknown **or** unpublished |
| `GET /blog/tags` | `{ tags: [{ slug, label, count }] }` over published posts |
| `GET /blog/media/{id}` | Image bytes |
| `GET /blog/config` | `{ media_base_url, environment }` — diagnostic |

`GET /blog/posts/{slug}` returns an **identical bare 404** whether the slug is
unknown or merely unpublished. A distinguishable response would leak that a
draft exists at that URL.

### Admin — `super_admin` only

| Method | Path | Body | Success |
|---|---|---|---|
| `GET` | `/admin/blog/posts?status=&q=&limit=&offset=` | — | `200` list, drafts **included**, `updated_at DESC` |
| `POST` | `/admin/blog/posts` | `CreateBlogPostBody` | `201` with `status: "draft"` |
| `GET` | `/admin/blog/posts/{id}` | — | `200` (by **id**, not slug) |
| `PATCH` | `/admin/blog/posts/{id}` | partial | `200` |
| `POST` | `/admin/blog/posts/{id}/publish` | `{ published_at? }` | `200` `status: "published"` |
| `POST` | `/admin/blog/posts/{id}/unpublish` | `{}` | `200` `status: "draft"` |
| `DELETE` | `/admin/blog/posts/{id}` | — | `204` |
| `POST` | `/admin/blog/media` | multipart, field **`file`** | `201 BlogMediaUpload` |

Every one returns **403** for any other role. The FE gate
(`<RequireAuth allowedRoles={["super_admin"]}>`) only hides UI — it is not a
security boundary.

### Behaviour the FE relies on

- **Create always yields a draft.** Publishing is a separate explicit call.
- **`publish` with no `published_at` stamps now.** Re-publishing an
  already-published post is **idempotent and must not move `published_at`** —
  that timestamp is the post's public identity (ordering, "posted on", feed
  position), and re-publishing after an edit is normal.
- **A future `published_at` is a scheduled post** — invisible to the public
  list, the detail route, and the tag counts until it arrives.
- **`unpublish`** returns to draft, keeps content, slug **and** `published_at`,
  so re-publishing restores the original date rather than silently re-dating.
- **`PATCH` cannot change `status` or `published_at`.** Those fields are absent
  from `UpdateBlogPostBody`, and the query layer filters to an explicit
  `WRITABLE_FIELDS` allowlist — so a stray key can never smuggle a post live.
- **Autosave**: the editor `PATCH`es a draft ~2s after typing stops. These
  writes are chatty by design — keep them cheap, and **keep usage/telemetry
  accounting out of them** or drafting a long post pollutes the usage ledger.
- **Tags** are trimmed, blank-dropped and de-duplicated preserving author
  order. They are **not** case-folded, so `Drilling` and `drilling` remain
  distinct chips.

### Error codes

| Code | HTTP | When |
|---|---|---|
| `blog_slug_taken` | 409 | Another post owns that slug |
| `blog_slug_empty` | 400 | Title/slug produced no usable slug |
| `blog_slug_immutable` | 409 | `slug` sent in a `PATCH` for a published post |
| `blog_post_not_found` | 404 | Unknown id |
| `blog_title_required` | 400 | Empty title on create/publish |
| `unsupported_media_type` | 400 | Upload MIME not in the allowlist |
| `media_too_large` | 400 | Upload over `BLOG_MEDIA_MAX_BYTES` |
| `blog_rate_limited` | 429 | Anonymous read cap per IP |

`media_too_large` is **400, not 413** — the contract pins it so the editor's
toast is identical whether the browser or the server rejected the file.

## Rate limiting

Public reads go through `enforce_blog_read_quota` (key `rl:blog:read:{ip}`),
`BLOG_READ_MAX_PER_IP_PER_HOUR` (default **600**). Generous on purpose: a real
visitor bouncing around the index plus a few posts (each pulling images) burns
requests quickly, so this exists to blunt scrapers, not to shape traffic.

**Fail-open**, like every limiter in [app/utils/rate_limit.py](../app/utils/rate_limit.py)
— a Redis outage must never take the public blog down. `/blog/config` is
deliberately exempt: it's what you reach for when the blog is already misbehaving.

## Configuration reference  ·  [app/config.py](../app/config.py)

| Setting | Default | Notes |
|---|---|---|
| `BLOG_MEDIA_BASE_URL` | `https://api.wellsynth.ai` | **Set per environment.** Baked into posts permanently. Blank falls back to the request origin — fine locally, wrong in prod. |
| `BLOG_MEDIA_MAX_BYTES` | `5242880` (5 MB) | Keep equal to the editor's client-side cap or the user gets a confusing rejection after a successful-looking upload. |
| `BLOG_READ_MAX_PER_IP_PER_HOUR` | `600` | Anonymous read cap. |

CORS needs no blog-specific change: `CORS_ALLOW_ORIGINS` already covers
`wellsynth.ai`, `www.` and `app.`.

## Testing  ·  [tests/test_blog.py](../tests/test_blog.py)

40 route-level tests with the DB and storage boundaries mocked, matching the
style in [Testing](09-testing.md). The ones that encode security rather than
behaviour:

- anonymous list/detail succeed with **no `Authorization` header**
- an unpublished slug is a plain 404, indistinguishable from unknown
- every admin route 403s for a non-super_admin (parametrized over all 8)
- `_is_anonymous("/blog/…")` is true and `_is_anonymous("/admin/blog/…")` is false
- `OPTIONS` preflight never 401s
- media responses carry `nosniff` + `sandbox` CSP

The role gate is exercised through a real `AuthMiddleware`-shaped user
injection rather than `dependency_overrides`, because each
`Depends(require_role(...))` is a distinct closure that overrides cannot reach.

## Runbooks

### Author and publish a post

1. Sign in as super_admin → `/admin/blog`.
2. `curl https://<api-host>/blog/config` and confirm `media_base_url` matches
   the environment **before** uploading images.
3. Write, upload images through the editor (never paste a hand-written image
   URL — the upload endpoint is what mints a resolvable one).
4. Publish. It is live immediately; no rebuild.

### Content does NOT move between environments

Blog posts are **data**, not code. `git push` ships the routes and the
migration; it does not ship a single post. A post authored locally stays
local, and its image URLs point at `localhost:8080`, which is unreachable from
anywhere else.

To get content into prod, **author it in prod**. That is almost always faster
and safer than migrating, because migrating means moving three things in step:
the `blog_posts` rows, the `blog_media` rows, the files under
`<storage_root>/blog/`, **and** rewriting every media URL inside every
`body_markdown`. If you must migrate, rewrite the URLs as part of the same
transaction that inserts the rows, and verify each post renders before
publishing it.

### An image renders broken

1. Open the image URL directly. `404` → the row or the file is missing;
   wrong host → the post was authored against a different environment.
2. `curl <api>/blog/config` — if `media_base_url` is wrong, the container has
   stale config. Restart it. **Already-uploaded images keep the old URL**;
   re-upload them.
3. Check the markdown: a bare `![image](image)` means someone pasted a
   reference instead of uploading. Only `POST /admin/blog/media` mints a
   resolvable URL.

### Apply the schema to a new environment

```bash
alembic upgrade head        # 016 → 017
python -m app.tools.check_schema_parity
```

## Known limitations

- **Client-rendered posts (FE).** The static export ships one `/blog/_.html`
  shell and SWA rewrites `/blog/*` onto it; per-post `<title>`, OG tags and
  `BlogPosting` JSON-LD are written at runtime. **Google renders JS so posts
  index normally, but LinkedIn/X/Slack do not** — shared links show the generic
  blog card. Fixing it means prerendering at build time, which needs a
  publish→rebuild webhook at the Azure DevOps pipeline. Deliberately deferred:
  it buys link previews only, and the pipeline trigger does not exist yet.
- **No revisions/audit.** `author_user_id` records who created a post; edits
  are not versioned.
- **No scheduled-publish worker.** A future `published_at` works because reads
  filter on `published_at <= now()`, not because anything wakes up to publish
  it. Correct, but it means nothing fires on the transition (no notification,
  no cache warm).
