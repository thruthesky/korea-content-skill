---
name: list-banners
description: "List banner ads at a position on a Korea SNS sub-site. Admin API key required. Example: '/korea:list-banners apple.withcenter.com header'. Use for listing banners, viewing banner ads, reviewing active banners, auditing banner schedule."
---

# /korea:list-banners — List Banner Ads

List banners at a given position on a sub-site. Admin-only; results include every banner (active, expired, and dated-but-not-yet-live) that hasn't been soft-deleted.

## Command Format

```
/korea:list-banners <sub-site domain> <position> [category_id]
```

- `<position>` — one of `header`, `sidebar`, `forum`.
- `[category_id]` — required when `<position>` is `forum`; ignored otherwise.

## Usage Examples

```
/korea:list-banners apple.withcenter.com header
/korea:list-banners bangphil.com sidebar
/korea:list-banners apple.withcenter.com forum 12
```

## Execution Procedure

### Step 1: Validate required parameters

| Information | Required | Description |
|-------------|----------|-------------|
| **Sub-site domain** | **O** | Admin's sub-site |
| **Position** | **O** | `header`, `sidebar`, `forum` |
| **Category id** | O (when position=forum) | Numeric forum category id |

**If missing / invalid:**
```
Usage: /korea:list-banners <sub-site domain> <position> [category_id]
Position must be one of: header, sidebar, forum.
For position=forum, a category_id is required.
```

### Step 2: Resolve base URL + category if needed

If the user gave a category name instead of an id, resolve it via:

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  categories --site-id {SITE_ID}
```

(You can look up `{SITE_ID}` via `sites`.) Then use the matching category's id for `--category-id`.

### Step 3: List

```bash
# Header / sidebar
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-list --position header

# Forum — category required
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-list --position forum --category-id 12
```

### Step 4: Format the result for the user

For every banner in `data[]`, show one row:

```
#{id} | {position}/{banner_type} | {title}
       {begin_at} → {end_at}   [ACTIVE | SCHEDULED | EXPIRED | NO-DATES]
       click_url: {click_url}   click_count: {click_count}
```

Active-status rule: `begin_at` and `end_at` must both be set, and `NOW()` must sit in that range. Otherwise mark the row `SCHEDULED` (begin in the future), `EXPIRED` (end in the past), or `NO-DATES` (either timestamp null).

If the list is empty, say so explicitly: `No banners at {position} on {sub-site}.`

Typical errors to relay: `401 Unauthorized.` (no/bad key), `403 Forbidden.` (not an admin of this sub-site), `422 position parameter is required.`, `422 category_id is required for forum position.`, `404 Site not found.` (base URL didn't resolve to a known sub-site).

## Notes

- The list is **admin-scoped** — it includes expired and soft-deleted-never banners too, sorted by `sort_order ASC, id DESC`. The **active-render** sort (duration-weighted) is a separate server-side query that this endpoint does not expose.
- Header / sidebar ignore `banner_type` in rendering — you may still see mixed types in the list, but they all render the same way (image-only).
- Each returned row also carries two arrays that are worth surfacing if the user is auditing: `attachments` (admin-internal files) and `advertiser_attachments` (public downloads on `/ad/show`).
