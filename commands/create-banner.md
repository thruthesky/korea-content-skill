---
name: create-banner
description: "Create a banner ad on a Korea SNS sub-site. Admin API key required. Example: '/korea:create-banner apple.withcenter.com Header banner for Spring Sale, runs April 20 to May 20, click goes to example.com'. Use for creating banners, banner ads, header banner, sidebar banner, forum banner, between-post ad."
---

# /korea:create-banner — Create a Banner Ad

Create a banner on a Korea SNS sub-site. Banner CRUD is **admin-only** and gated per sub-site by the `Host` header, so the caller must pass an admin API key **for that sub-site**.

## Command Format

```
/korea:create-banner <sub-site domain> <prompt>
```

**Both parameters are required.** If either is missing, abort and guide the user.

## Usage Examples

```
/korea:create-banner apple.withcenter.com Header banner titled "Spring Sale", runs 2026-04-20 to 2026-05-20, click to https://example.com/spring
/korea:create-banner bangphil.com Forum small-type banner in Restaurants category, title "Yakimix Buffet", subtitle "Cebu IT Park", runs April 20 – May 20
/korea:create-banner cherry.withcenter.com Sidebar banner for a new café, no dates yet
```

## Execution Procedure

### Step 1: Validate required parameters

| Information | Required | Description |
|-------------|----------|-------------|
| **Sub-site domain** | **O** | First token, must be the admin's sub-site (e.g. `apple.withcenter.com`). Main domain `withcenter.com` will 403. |
| **Prompt** | **O** | Natural-language description of the banner (title, position, dates, URL, etc.) |

**If either is missing:**
```
Please provide a sub-site domain and a prompt.
Usage: /korea:create-banner <sub-site domain> <prompt>
Example: /korea:create-banner apple.withcenter.com Header banner for Spring Sale, April 20 – May 20
```

### Step 2: Check the API key

Confirm the caller already has an admin API key for that sub-site. If not:
```
An admin API key for <sub-site> is required. Please provide it before continuing.
```
Do not try to log in on the main domain — admin flows must be against the sub-site host.

### Step 3: Resolve the sub-site base URL

The base URL **must** point at the sub-site. If the prompt used a bare name ("apple"), map it to the host via `sites`:

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  sites
```

Use the resolved subdomain as `{SUBSITE}` for every subsequent call. If no match, show the available sites and abort.

### Step 4: Extract fields from the prompt

From the prompt, pull out:

| Field | Required? | How to derive |
|-------|-----------|---------------|
| `title` | **O** | Quoted label or inferred subject |
| `position` | **O** | Default to `header` unless the prompt says sidebar/forum |
| `banner_type` | if forum | `large` for image-only, `small` for thumbnail+text, `between` for inter-post ads |
| `category_id` | if forum | Resolve from `categories --site-id {ID}` |
| `begin_at` / `end_at` | recommended | Absolute ISO (`2026-04-20 00:00:00`). **If missing, warn the user: a banner with no dates will never render.** |
| `click_url` | optional | URL in the prompt, if any |
| `subtitle` | optional (small) | Sub-line for `small` / `between` types |
| `content`, `notes` | optional | Rich content for `/ad/show`, and admin-only notes |
| `between_interval` | optional (between) | "Show every N posts"; default 5 |
| 7× `contact_*` | optional | Advertiser contacts (telegram/phone/kakao/email/facebook/whatsapp/viber) |

If the prompt mentions a forum banner without a category, ask the user which category. If it mentions no dates, ask before proceeding — an always-inactive banner is rarely intended.

### Step 5: Source and upload the banner image

Every banner needs a display image. Pick the source in this priority order:

1. **User-supplied local path** — use it as-is.
2. **Picsum placeholder (default for tests / bulk seeding)** — download a random image from `https://picsum.photos/<w>/<h>` to a local temp file, then upload. This is the default whenever the user is seeding banners for testing and hasn't provided an image.
3. **Internet search (opt-in, "interesting/fun" imagery)** — only when the user explicitly asks for real-looking or topical imagery. Use `WebSearch` to find a royalty-free candidate (Unsplash, Pexels, Wikimedia Commons, or `images.unsplash.com/*`), `curl` it to a local file with a realistic `User-Agent`, then upload.

In every path, **download to a local file first**, then hand the path to `korea_api.py upload`. Never pass a remote URL to the API — the server only accepts multipart uploads.

#### Option A — Picsum (default, test-friendly)

Picsum serves random, license-free JPEGs at any size. Pick dimensions that match the position:

| Position | Recommended size |
|----------|------------------|
| `header`, `sidebar` | `400x300` (or `600x300` for wide header) |
| `forum` `large` | `400x400` (rendered at 100×100 square) |
| `forum` `small` | `256x168` (rendered at 64×42) |
| `forum` `between` | `800x400` (wide inter-post card) |

```bash
# Download a unique random picsum image (append ?random=N so repeated calls differ)
N=$RANDOM
TMP="/tmp/banner-${N}.jpg"
curl -sL "https://picsum.photos/400/300?random=${N}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -o "${TMP}"

# Upload to the sub-site
python3 .claude/skills/content/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  upload --file "${TMP}"
# → { "data": { "id": 101, ... } }
```

When seeding **N** banners in a single run, loop and capture each upload id so the banners don't all share one image — reusing a single upload id is allowed by the API but defeats the point of test coverage:

```bash
for i in $(seq 1 8); do
  tmp="/tmp/banner-$$-$i.jpg"
  curl -sL "https://picsum.photos/400/300?random=$$-$i" -o "$tmp" -H "User-Agent: KoreaSNS-CLI/1.0"
  python3 .claude/skills/content/scripts/korea_api.py --api-key "{KEY}" \
    --base-url "https://{SUBSITE}/api/v1" upload --file "$tmp"
done
```

Picsum occasionally redirects — always use `curl -L`. If picsum is unreachable, fall back to a locally generated placeholder (e.g. a tiny solid-color PNG) rather than failing the whole seed run.

#### Option B — Internet search (interesting/fun, opt-in)

Use this only when the user asks for real-looking or topical imagery (e.g. "find a fun café photo"). Workflow:

1. Run `WebSearch` with the banner topic + `"royalty free"` or a known-licensed source (`site:unsplash.com`, `site:pexels.com`, `site:commons.wikimedia.org`).
2. Pick a direct image URL (must end in `.jpg`, `.jpeg`, `.png`, or `.webp`; Unsplash's `images.unsplash.com/photo-...` URLs are direct).
3. Download with `curl -L` and a realistic `User-Agent` — some CDNs block empty UAs.
4. Upload the local file the same way as Option A.

```bash
TMP="/tmp/banner-fun.jpg"
curl -sL "https://images.unsplash.com/photo-xxxxxxxx?w=800" \
  -H "User-Agent: Mozilla/5.0 (compatible; KoreaSNS-CLI/1.0)" \
  -o "${TMP}"

python3 .claude/skills/content/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  upload --file "${TMP}"
```

**Source hygiene:** prefer explicitly free-to-use libraries (Unsplash, Pexels, Pixabay, Wikimedia Commons with a compatible license). Do not hotlink — the API rejects remote URLs and hotlinking external CDNs in production banners will break when the source rotates. Download, upload, done.

Capture the returned `id` for `--upload-ids`. If the user insists on no image, confirm once (header/sidebar with no image render an empty slot).

### Step 6: Create the banner

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-create \
  --title "Spring Sale" --position header \
  --begin-at "2026-04-20 00:00:00" --end-at "2026-05-20 00:00:00" \
  --click-url "https://example.com/spring" \
  --upload-ids "101"
```

For forum banners add `--banner-type` and `--category-id`. Advertiser contacts go through `--contact-telegram`, `--contact-phone`, etc.

### Step 7: Report the result

Tell the user:
- The created banner id and position/type
- Whether it is **currently active** (today within `begin_at`…`end_at`) or not
- A reminder of the sub-site it was created on

On server error, relay the `message` field verbatim. Common ones: `title is required.` (422), `category_id is required for forum position.` (422), `Forbidden.` (403 — wrong sub-site), `Site not found.` (404 — base URL didn't resolve).

## Notes

- **Admin key is per sub-site.** Passing `https://withcenter.com/api/v1` as base URL with a sub-site admin key returns 403.
- **Attachments:** `--upload-ids` is the display image; `--attachment-ids` is admin-internal (receipts); `--advertiser-attachment-ids` is public on `/ad/show`.
- **Type is ignored for header/sidebar** — they always render as a single image. Set `banner_type` only for forum banners.
- Full field / enum / error reference: [../references/api-banners.md](../references/api-banners.md).
