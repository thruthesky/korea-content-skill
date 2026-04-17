# Banner Ads API (admin CRUD + public click tracking)

> Parent document: [SKILL.md](../SKILL.md) — see [api-auth.md](api-auth.md) for authentication and [api-content.md](api-content.md) for the file-upload flow this doc reuses.

## Core Concepts

Banner ads are managed exclusively by sub-site administrators. Every endpoint except the public click tracker requires an admin API key **for the sub-site the caller is targeting**. The system has no "active toggle" — a banner is live only while the current time sits between `begin_at` and `end_at` (both must be set). `DELETE` is soft (sets `deleted_at`).

There are three **positions** where banners render, and three **types** controlling the visual treatment. Only the `forum` position honors the full type matrix — `header` and `sidebar` render every banner as an image regardless of type.

## Core Logic

- **Admin check is per sub-site.** `requireSiteAdmin()` resolves the caller's site from the HTTP `Host` header, so the base URL **must** point to the sub-site you want to manage (e.g. `https://apple.withcenter.com/api/v1`, not `https://withcenter.com/api/v1`). A main-domain call with a sub-site admin key returns `403`.
- **Active status is date-gated.** A banner with `begin_at = NULL` or `end_at = NULL` is never active. Rendering queries filter on `deleted_at IS NULL AND begin_at <= NOW() AND end_at >= NOW()`. There is no `is_active` flag exposed in the API.
- **Forum banners need a category.** When `position = forum`, `category_id` is required on both create and list. For `header` / `sidebar`, `category_id` is ignored (leave it `null`).
- **Attachments are split into three roles** (see [§ Attachments](#attachments)) — display image, admin-internal files, advertiser-facing downloads — and each is uploaded via the generic `POST /files/upload` endpoint, then passed as an array of upload IDs.
- **Click tracking is public.** `POST /banners/{id}/click` requires no auth and is called via `navigator.sendBeacon()` from the client side.

## Base URL

```
# Main domain — banner endpoints will 403 for sub-site admins
https://withcenter.com/api/v1

# Sub-site (use this one)
https://<subdomain>.withcenter.com/api/v1
```

**Critical:** every admin banner call must hit the sub-site host that the admin key belongs to. Mismatch → `403 Forbidden`.

---

## Position × Type Matrix

The combination of `position` and `banner_type` decides where and how the banner renders.

| `position` | `banner_type` | `category_id` | Where it renders | Notes |
|------------|---------------|---------------|------------------|-------|
| `header`   | any (ignored) | — (null)      | Two slots left/right of the search bar at the top of the site; 3.5 s rotation, shared pool across both slots | Canonical header ad. `banner_type` is stored but ignored by the renderer. |
| `sidebar`  | any (ignored) | — (null)      | Floating left/right columns on wide viewports, 120×120 squares, interleaved (idx 0 → left, 1 → right, …) | Canonical sidebar ad. `banner_type` is stored but ignored by the renderer. |
| `forum`    | `large`       | **required**  | Image grid above the post list on the category page; 100×100 squares, auto-wrap | Image-only (no text). Use for brand/hero ads inside a category. |
| `forum`    | `small`       | **required**  | Vertical list below the large grid; 64×42 thumbnail + title + subtitle | Use when you need headline text alongside the image. |
| `forum`    | `between`     | **required**  | Injected between posts in the category list every `between_interval` posts | `between_interval` defaults to `5`, i.e. one ad after every 5th post. |
| `forum`    | —             | missing       | N/A                                                                 | Server returns `422`. |
| `header` / `sidebar` | `small` or `between` | —   | Silently treated as `large` (type is ignored)                       | Avoid — confuses admins later reading the list. |

**Sort order of active banners:** `(end_at - begin_at) DESC, id DESC` — longer campaigns sort first (duration-weighted rotation). `sort_order` is stored on the row but is not used by the active-render queries.

## Enum Reference

| Enum | File | Values |
|------|------|--------|
| `BannerPosition` | `src/Enums/BannerPosition.php` | `header`, `sidebar`, `forum` |
| `BannerType`     | `src/Enums/BannerType.php`     | `large`, `small`, `between` |

Send the raw string value in JSON (`"position": "header"`, `"banner_type": "large"`). The server calls `Enum::from()` and will `ValueError` on an unknown string — the caller sees a generic `500`, so validate client-side.

---

## 1. List banners

### GET /admin/banners — List banners at a position

**Authentication**: Required (site administrator)

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `position` | string | **O** | `header`, `sidebar`, or `forum` |
| `category_id` | int | O (when `position=forum`) | Forum category ID. Ignored for `header`/`sidebar`. |

```bash
# Header pool
curl -s "https://apple.withcenter.com/api/v1/admin/banners?position=header" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0"

# Forum pool for category 12
curl -s "https://apple.withcenter.com/api/v1/admin/banners?position=forum&category_id=12" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0"
```

**Success response (200)** — includes both `attachments` (admin-internal) and `advertiser_attachments` (public) for every banner:

```json
{
  "data": [
    {
      "id": 42,
      "site_id": 7,
      "category_id": null,
      "position": "header",
      "banner_type": "large",
      "title": "Spring Sale",
      "subtitle": null,
      "image_url": "/uploads/7/banner-hero.jpg",
      "click_url": "https://example.com/spring",
      "content": null,
      "notes": "Paid by ACME Corp, receipt in attachments",
      "sort_order": 0,
      "between_interval": 5,
      "click_count": 341,
      "begin_at": "2026-04-20 00:00:00+00",
      "end_at":   "2026-05-20 00:00:00+00",
      "contact_telegram": null,
      "contact_phone": "+82-10-1234-5678",
      "contact_kakao": null,
      "contact_email": "sales@example.com",
      "contact_facebook": null,
      "contact_whatsapp": null,
      "contact_viber": null,
      "created_at": "2026-04-17 09:00:00+00",
      "updated_at": "2026-04-17 09:00:00+00",
      "deleted_at": null,
      "attachments": [],
      "advertiser_attachments": []
    }
  ]
}
```

**Errors**: `401` no/invalid API key · `403` not an admin of this sub-site · `422` `position` missing, or `position=forum` without `category_id`.

---

### GET /admin/banners/{id} — Get a single banner

**Authentication**: Required (site administrator)

```bash
curl -s https://apple.withcenter.com/api/v1/admin/banners/42 \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0"
```

**Success (200)**: same shape as a single list item (includes `attachments`, `advertiser_attachments`).

**Errors**: `401`, `403`, `404 "Banner not found."` (also returned if the banner belongs to a different sub-site).

---

## 2. POST /admin/banners — Create a banner

**Authentication**: Required (site administrator)

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `title` | string | **O** | — | Admin-visible label; also shown on `small` and `between` renders |
| `position` | string | **O** | — | `header`, `sidebar`, `forum` |
| `category_id` | int | O (when `position=forum`) | null | Forum category |
| `banner_type` | string | X | `large` | `large`, `small`, `between`. Only meaningful for `position=forum` |
| `subtitle` | string | X | null | Shown on `small` / `between` renders |
| `click_url` | string | X | null | Destination on click; opened in a new tab |
| `content` | string | X | null | Rich text for the `/ad/show` detail page |
| `notes` | string | X | null | Admin-only; never rendered to visitors |
| `sort_order` | int | X | 0 | Stored but **not** used by render ordering — display sort is duration-weighted |
| `between_interval` | int | X | 5 | For `banner_type=between`: show this ad after every N posts |
| `begin_at` | string | X | null | ISO timestamp. Banner is active only if both `begin_at` and `end_at` are set and `NOW()` is in range |
| `end_at` | string | X | null | ISO timestamp |
| `contact_telegram` | string | X | null | Advertiser Telegram handle; empty string is stored as null |
| `contact_phone` | string | X | null | Advertiser phone |
| `contact_kakao` | string | X | null | Kakao ID |
| `contact_email` | string | X | null | Email |
| `contact_facebook` | string | X | null | Facebook profile / page URL |
| `contact_whatsapp` | string | X | null | WhatsApp number |
| `contact_viber` | string | X | null | Viber number |
| `upload_ids` | int[] | X | `[]` | Sets the **display image**. Attached to target `banner_ads`. Upload via `POST /files/upload` first. |
| `attachment_ids` | int[] | X | `[]` | **Admin-internal** files (e.g. payment receipts). Attached to target `banner_ad_attachments`. Never shown to public visitors. |
| `advertiser_attachment_ids` | int[] | X | `[]` | **Public** files shown on the `/ad/show` detail page. Attached to target `banner_ad_advertiser_attachments`. |

```bash
curl -s -X POST https://apple.withcenter.com/api/v1/admin/banners \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{
        "title": "Spring Sale",
        "position": "header",
        "begin_at": "2026-04-20 00:00:00",
        "end_at":   "2026-05-20 00:00:00",
        "click_url": "https://example.com/spring",
        "upload_ids": [101],
        "contact_email": "sales@example.com",
        "notes": "Paid by ACME Corp"
      }'
```

**Success response (201)**: full banner row including the three attachment arrays; `site_id` is set from the sub-site context automatically.

**Errors**:
- `422 "title is required."` — missing title
- `422 "position is required."` — missing position
- `422 "category_id is required for forum position."` — forum banner without category
- `403 "Forbidden."` — caller is not an admin of this sub-site
- `401 "Unauthorized."` — no/invalid API key

**Business rules**:
- A banner without both `begin_at` and `end_at` is **stored but never rendered** (use it as a draft mechanism; renderer skips rows with nullable dates).
- `site_id` comes from `SiteContext::getId()` — **never** pass it in the body (no effect; IDOR-safe).

---

## 3. PUT /admin/banners/{id} — Update a banner

**Authentication**: Required (site administrator)

Partial update with `array_key_exists` semantics: **omit a key to leave the field untouched**, set a nullable field to `null` or `""` to clear it. `is_active` is not in the whitelist and is silently ignored.

Accepted fields (same as create, minus `is_active`): `title`, `subtitle`, `click_url`, `content`, `notes`, `sort_order`, `between_interval`, `begin_at`, `end_at`, `position`, `banner_type`, `category_id`, 7× `contact_*`, `upload_ids`, `attachment_ids`, `advertiser_attachment_ids`.

```bash
# Rename only
curl -s -X PUT https://apple.withcenter.com/api/v1/admin/banners/42 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{"title": "Spring Mega Sale"}'

# Clear the phone contact while extending the run
curl -s -X PUT https://apple.withcenter.com/api/v1/admin/banners/42 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{"contact_phone": "", "end_at": "2026-06-01 00:00:00"}'
```

**Success (200)**: full updated banner row.

**Errors**: `401`, `403`, `404 "Banner not found."` (also when the row is in another sub-site).

**Gotchas**:
- Passing `upload_ids` replaces the display image — to keep it, omit the key.
- Passing `attachment_ids` / `advertiser_attachment_ids` appends new files to the banner's attachment targets (see [`BannerAdService::update`](../../../../src/Services/BannerAdService.php)); to remove a file, delete it via `DELETE /files/{id}`.
- Setting `begin_at` or `end_at` to `""` normalizes to `null` (disables the banner).

---

## 4. DELETE /admin/banners/{id} — Delete a banner (soft)

**Authentication**: Required (site administrator)

```bash
curl -s -X DELETE https://apple.withcenter.com/api/v1/admin/banners/42 \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0"
```

**Success (200)**: `{ "data": { "deleted": true } }`

**Errors**: `401`, `403`, `404 "Banner not found."`

**Business rule**: Soft delete — sets `deleted_at`, row is excluded from every active and admin list query. There is no API to restore; restore by `UPDATE banner_ads SET deleted_at = NULL` in the DB.

---

## 5. POST /banners/{id}/click — Public click tracker

**Authentication**: **None** (public)

No body. Increments `click_count` by 1. Called from the client side via `navigator.sendBeacon()` when a visitor clicks the banner; do not wire this into server-side admin flows.

```bash
curl -s -X POST https://apple.withcenter.com/api/v1/banners/42/click \
  -H "User-Agent: KoreaSNS-CLI/1.0"
```

**Success (200)**: `{ "data": { "clicked": true } }`

No 404 guard — an unknown `id` silently no-ops (the repository uses `UPDATE ... WHERE id = ?`).

---

## Attachments

Banners use three distinct file-attachment "targets", all uploaded through the shared `POST /files/upload` endpoint (documented in [api-system.md](api-system.md)).

| Input field | Storage target | Visibility | Typical use |
|-------------|----------------|------------|-------------|
| `upload_ids` | `banner_ads` | Renders as the banner's display image | The image a visitor actually sees |
| `attachment_ids` | `banner_ad_attachments` | Admin-only | Bank transfer receipts, invoices, sign-off PDFs |
| `advertiser_attachment_ids` | `banner_ad_advertiser_attachments` | Public on `/ad/show` | Brochures, coupons, downloadable catalogs |

Workflow is identical to posts: upload first, then pass the returned `id`s.

```bash
# Step 1 — upload on the sub-site (not the main domain)
curl -s -X POST https://apple.withcenter.com/api/v1/files/upload \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -F "file=@/path/to/banner.jpg"
# → { "data": { "id": 101, ... } }

# Step 2 — create the banner pointing at that upload
curl -s -X POST https://apple.withcenter.com/api/v1/admin/banners \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{"title":"…","position":"header","upload_ids":[101], "begin_at":"…","end_at":"…"}'
```

---

## Common errors summary

| Status | Message (server) | Cause |
|--------|------------------|-------|
| `401`  | `"Unauthorized."` | No `Authorization: Bearer` header or key invalid |
| `403`  | `"Forbidden."` | API key's user is not a site admin of the sub-site the base URL resolves to |
| `404`  | `"Banner not found."` | Wrong id, wrong sub-site, or soft-deleted |
| `404`  | `"Site not found."` | `Host` header didn't resolve to a known sub-site |
| `422`  | `"title is required."` / `"position is required."` | Missing required field on create |
| `422`  | `"category_id is required for forum position."` | `position=forum` without `category_id` on create or list |
| `422`  | `"position parameter is required."` | `GET /admin/banners` without `position` query param |
