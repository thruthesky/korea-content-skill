---
name: content
description: "Complete content management via Korea SNS (withcenter.com) REST API. API key based authentication (Authorization: Bearer). Supports user registration/login/profile updates, post CRUD, comment CRUD, file (image) uploads, site/category lookup and management, likes/bookmarks/reactions, notifications, search, and sub-site admin banner ads (header/sidebar/forum banners with begin_at/end_at scheduling). A complete API guide for external programs, AI, and software to create, update, and delete content on Korea SNS. Use when Claude performs the following tasks: (1) Korea SNS registration/login/API key retrieval (2) User profile update/avatar upload (3) Post create/update/delete/list (4) Comment create/update/delete (5) Upload images and files and attach them to posts/comments (6) Site list/category tree lookup (7) Toggle like/bookmark/reaction (8) Notifications lookup/search (9) withcenter.com API calls (10) Automatic API documentation lookup (GET /docs) (11) Banner ads CRUD on a sub-site (header, sidebar, forum large/small/between banners — requires sub-site admin API key). Keywords: Korea SNS, withcenter, post, posts, writing, post registration, post update, post deletion, comment, comments, registration, login, profile, avatar, file upload, image, category, site, like, bookmark, reaction, notification, search, API, api_key, banner, banner ad, banners, header banner, sidebar banner, forum banner, between-post ad, banner_type, banner position, banner schedule, begin_at, end_at, ad scheduling"
---

# Korea SNS — Complete Content Management Skill

A skill that performs all content tasks such as user management, post/comment CRUD, file upload, and site/category management through the Korea SNS (withcenter.com) REST API.

## API Basics

| Item | Value |
|------|-------|
| **Base URL** | `https://withcenter.com/api/v1` |
| **Authentication** | `Authorization: Bearer {API_KEY}` header |
| **API key format** | `{user_id}-{md5_hash}` (e.g., `4-a1b2c3d4e5f6...`) |
| **Request format** | JSON: `Content-Type: application/json`, files: `multipart/form-data` |
| **Success response** | Single: `{ "data": {...} }`, list: `{ "data": [...], "meta": {...} }` |
| **Error response** | `{ "message": "error message" }` |
| **User-Agent** | `User-Agent: KoreaSNS-CLI/1.0` is required to bypass Cloudflare WAF blocking |

Three ways to pass the API key (in priority order):
1. **Authorization header** (recommended): `Authorization: Bearer {API_KEY}`
2. **api_key cookie**: `Cookie: api_key={API_KEY}`
3. **Query parameter**: `?api_key={API_KEY}`

## Automatic API Documentation Lookup

When unsure which APIs are available, call the following endpoint to get the list of APIs:

```bash
# Fetch full API documentation
curl -s https://withcenter.com/api/v1/docs \
  -H "User-Agent: KoreaSNS-CLI/1.0"

# Filter by category (auth, user, post, comment, file, site, category, notification, search)
curl -s "https://withcenter.com/api/v1/docs?category=post" \
  -H "User-Agent: KoreaSNS-CLI/1.0"
```

**Always check API documentation before starting work to identify available endpoints.**

## Workflow

### Step 1: Obtain an API key

If the user provides an API key directly, use it.
If there is no API key but an email/password is available, log in to obtain the API key.
If there is no account, register first and automatically obtain the API key.

**Important: When working on a subsite (e.g., apple.withcenter.com), always set `--base-url` to the subsite URL.**

```bash
# Register (on subsite — registration is not allowed on the main site)
python3 skills/korea/scripts/korea_api.py --api-key "" \
  --base-url "https://apple.withcenter.com/api/v1" \
  register --email "user@example.com" --password "pass123" --display-name "NewUser"

# Login
python3 skills/korea/scripts/korea_api.py --api-key "" \
  --base-url "https://apple.withcenter.com/api/v1" \
  login --email "user@example.com" --password "pass"
```

### Step 2: Check site/category

Check the target site and category before writing a post.

```bash
# List sites
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" sites

# Fetch category tree (site ID required)
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" categories --site-id 1
```

### Step 2.5: AI Pre-flight Topic Workflow (Plan → Reserve → Generate → **Score** → Submit)

**When AI is generating content**, do not waste tokens by drafting first and being rejected later, and do not pollute the site with low-quality posts. Always run this **5-step handshake** with the server. Steps 1–3 are cheap API calls. Step 4 is the expensive LLM/web-search work — it runs **only after** a successful reservation. Step 5 is a self-evaluation gate that **must pass at ≥ 90/100** before submission.

1. **Plan** — Call `topic-coverage` to see which `topic_slug`s the current user has already used.
   ```bash
   python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" \
     topic-coverage [--category-id 3] [--per-page 100]
   ```
2. **Check (optional)** — Batch-check candidate slugs before reserving. Returns `available` / `taken_by_post` / `reserved_active`.
   ```bash
   python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" \
     topic-check --topic-slugs "ph-cebu-diving,ph-palawan-elnido,ph-bohol-tarsier"
   ```
3. **Reserve** — Atomically lock ONE slug for up to TTL minutes (default 30, range 1–120). If 409, pick the next candidate.
   ```bash
   python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" \
     topic-reserve --topic-slug "ph-cebu-diving" [--category-id 3] [--ttl-minutes 30]
   ```
4. **Generate** — ONLY after a successful reservation, do the expensive work:
   - Gather **≥ 20 distinct web sources**
   - **Download ≥ 3 images locally** (curl with realistic User-Agent), then **upload each to the server** via `upload`, capture the returned `id` and server URL — never hotlink external images
   - Draft the article in **Markdown** (≥ 5,000 words / ≥ 9,000 Korean chars), using the **full Markdown syntax** (headings, bullet + numbered lists, table, bold + italic, blockquote, inline link, horizontal rule, inline code, image)
   - Sprinkle **≥ 15 emojis** through the body; every `##` heading gets a leading topic-matching emoji
   - End the body with the **mandatory 📍 방문 정보 metadata table** containing country, region, city, full address, lat/lon (≥ 4 decimals), name, contact (price + hours optional)
5. **Score & Submit** — Apply the **Content Quality Score rubric** in [references/content-quality-score.md](references/content-quality-score.md). Submit **only if the total score ≥ 90/100 AND every one of the 32 gates passes** (G1–G32). Otherwise revise (max 2 retries) or abandon the reservation.
   ```bash
   # only after rubric passes — note --upload-ids must list every uploaded image:
   python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" \
     create --title "..." --content "$BODY_MARKDOWN" --category-id 3 \
     --topic-slug "ph-cebu-yakimix-buffet" --reservation-id 889 \
     --upload-ids "124,125,126,127,128"
   ```

**The 90/100 quality gate AND 32 hard gates are mandatory.** A reserved-but-low-quality post is worse than no post — it consumes the slug permanently for this user (the reservation locks the slug after submission). If three drafts (1 original + 2 revisions) cannot pass every gate AND reach 90, **do not submit**; let the reservation expire and pick a different topic. **Never invent addresses, coordinates, or phone numbers** — fabrication forces a 0 in Accuracy and instantly fails the rubric.

> ⏱️ **TTL hint**: Long-form posts (5,000+ words + image upload) often exceed the default 30-minute TTL. Use `topic-reserve --ttl-minutes 90` (max 120) when reserving for full place guides.

**Rules of the system:**
- Uniqueness is enforced **per user**: user A and user B can each post the same topic once; user A cannot post the same `topic_slug` (or the same normalized title+content) twice.
- Posts **without** `topic_slug` are treated as normal human posts and are **not** subject to duplicate prevention or the score gate (e.g. a user re-listing the same sale post every day is allowed).
- `content_hash` is computed server-side (SHA-256 of NFC-normalized title+content) and is only stored/checked when `topic_slug` is present.
- Reservations expire automatically after TTL; the next `topic-reserve` call for the same slug cleans up expired rows before inserting a new one.
- `409 Conflict` means the slug or the content hash is already in use for this user — catch it, pick a different topic, retry.

Only when topic-reserve returns 201 should the AI spend real tokens on web search and drafting. Only when the rubric returns total ≥ 90 should the AI call `POST /posts`.

#### Quality Score — quick summary

100 points across 6 quality categories, gated by **32 hard requirements** (G1–G32). Full rubric, image/metadata workflow, and revision guidance: [references/content-quality-score.md](references/content-quality-score.md).

| Category | Max | What it measures |
|----------|----:|------------------|
| Originality & Uniqueness | 20 | Genuinely new framing/analysis vs. rehashed search results |
| Depth & Substance | 20 | Concrete numbers, comparisons, trade-offs vs. surface definitions |
| Accuracy & Verifiability | 15 | Sourced, current facts (incl. coordinates, address, phone) vs. fabrication |
| Structure, Markdown & Readability | 15 | Sections + full Markdown syntax + short paragraphs + visual rhythm |
| Reader Value | 20 | Tailored to Korean expat audience, KRW conversions, Maps link, actionable |
| Polish | 10 | No typos, natural Korean, no machine-translation artifacts, valid Markdown |

**Pass requirement:** `total ≥ 90` AND **all 32 gates pass**. The 32 gates cover:

| Group | Gates | Summary |
|-------|-------|---------|
| Content | G1–G7 | ≥ 20 sources · **≥ 5,000 words** (≥ 9,000 KO chars) · ≥ 5 sections · slug match · slug available · coherent language · **valid Markdown with full syntax** |
| Metadata (REQUIRED) | G8–G14 | 🏳️ Country · 🗺️ Region · 🏙️ City · 📮 Address · 📌 **Lat/Lon (≥ 4 decimals)** · 🏢 Name · 📞 Contact |
| Images | G15–G19 | ≥ 3 images · **download → upload → server URL** (no hotlinking) · descriptive alt text · ≥ 1 photo of the place · `--upload-ids` matches body URLs |
| Markdown syntax | G20–G28 | Headings ##/### · bullet **and** numbered list · table · **bold** + *italic* · blockquote · inline link · `---` rule · inline code · Markdown image syntax |
| Emoji & readability | G29–G32 | ≥ 15 emojis spread out · every `##` heading has a topic emoji · paragraphs ≤ 6 sentences (avg ≤ 4) · lists/tables/callouts every ~600 words |

**Optional metadata** (do not gate, include if known): 💰 Pricing, ⏰ Hours, 🌐 Website.

**On failure:**
1. Fix **gate failures first** — they are absolute (e.g. G2 word count, G15 images, G12 coordinates).
2. Identify `weak_areas` (quality categories below the "Good" band) and revise **only those** — do not rewrite from scratch on round 1.
3. Re-score.
4. Max 2 revisions; if still failing, abandon the reservation (let it expire) and pick another topic.

The AI must keep an internal scorecard JSON like:
```json
{ "word_count": 5421, "image_count": 5,
  "metadata": { "country":"Philippines","region":"Cebu Province","city":"Cebu City",
                "address":"123 Mango Ave, Lahug, Cebu City 6000",
                "lat":10.3157, "lon":123.8854,
                "name":"Sample Restaurant", "contact":"+63 32 123 4567" },
  "gates_passed": ["G1","G2","...","G32"],
  "scores": { "originality":18, "depth":17, "accuracy":14, "structure":13, "reader_value":19, "polish":9 },
  "total": 90, "pass": true, "weak_areas": [], "revision_round": 0 }
```
Inflating scores or fabricating metadata defeats the purpose. **Never invent addresses, coordinates, or phone numbers** — abandon the topic instead.

### Step 3: Execute the content task

Include `--base-url "https://<domain>/api/v1"` in every command.
In the examples below, `{BASE}` is a subsite URL such as `https://apple.withcenter.com/api/v1`.

```bash
# Post CRUD
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" create --title "Title" --content "Content" [--category-id 3]
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" update --id {ID} --title "New title" --content "New content"
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" delete --id {ID}
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" get --id {ID}
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" list [--page 1] [--per-page 10] [--category free]

# Comment CRUD
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" comment-create --post-id {ID} --content "Comment"
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" comment-update --comment-id {ID} --content "Updated"
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" comment-delete --comment-id {ID}

# Upload a file and attach it to a post
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" upload --file "/path/to/image.jpg"
# → Pass the returned upload ID through --upload-ids when creating a post/comment
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" create --title "Photo post" --content "Content" --upload-ids "10,11"

# Test/seed images with no user-supplied file:
#   1. Default → picsum.photos (curl -L to a local temp file, then upload the file)
#        curl -sL "https://picsum.photos/400/300?random=$RANDOM" -o /tmp/img.jpg
#   2. Opt-in for "interesting/fun" imagery → WebSearch a royalty-free source
#      (Unsplash / Pexels / Wikimedia Commons), curl -L to a local file, then upload.
#   Always download locally first — the /files/upload endpoint only accepts multipart, never a remote URL.
#   Full workflow + size table: commands/create-banner.md Step 5.

# Profile update
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" update-profile --display-name "New name" --bio "About me"

# Avatar upload
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" --base-url "{BASE}" upload-avatar --file "/path/to/avatar.jpg"
```

### Using curl directly (alternative)

Always include the `User-Agent` header to avoid Cloudflare WAF blocking.

```bash
# Create a post
curl -s -X POST https://withcenter.com/api/v1/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{"title": "Title", "content": "Content", "category_id": 3}'

# Upload a file
curl -s -X POST https://withcenter.com/api/v1/files/upload \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -F "file=@/path/to/image.jpg"

# Attach the uploaded file to a post
curl -s -X POST https://withcenter.com/api/v1/posts \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {API_KEY}" \
  -H "User-Agent: KoreaSNS-CLI/1.0" \
  -d '{"title": "Photo post", "content": "Content", "upload_ids": [10, 11]}'
```

## Data / Information Collection

When the user's prompt requires collecting information, research, facts, news, references, or background material before creating/updating content, you **MUST** gather data from a **minimum of 20 different sites (web pages)** before writing.

### Rules

1. **Minimum 20 sources**: Do not produce content until you have collected information from at least 20 distinct web pages. Fewer sources is not acceptable — expand the search until this threshold is met.
2. **Primary tools**: Use the `WebSearch` tool to discover candidate sources, and use the `WebFetch` tool to retrieve the full content of each page.
3. **Fallback to curl**: If `WebFetch` is unavailable, blocked, or fails, use `curl` via the Bash tool to fetch the page. Always include a realistic `User-Agent` header (e.g., `-H "User-Agent: Mozilla/5.0 (compatible; KoreaSNS-CLI/1.0)"`).
4. **Source diversity**: Prefer different domains over multiple pages from the same site. Aim for a mix of official/primary sources, news outlets, community discussions, and reference material relevant to the user's topic.
5. **Recency**: Prioritize recent sources when the topic is time-sensitive (news, prices, events, releases, etc.).
6. **De-duplicate**: If two pages contain substantially the same content (syndicated articles, mirrors), count them as a single source and keep searching.
7. **Record sources**: Keep track of the URLs you consulted so they can be cited or referenced in the final post/comment when appropriate.
8. **Synthesize, don't copy**: Combine and paraphrase findings into original content. Do not paste copyrighted text verbatim.

### Workflow

```
1. Parse the user's prompt → extract key topics / keywords / questions.
2. Run WebSearch with multiple query variations to build a candidate URL list (target ≥ 30 URLs so that after filtering you still have ≥ 20 usable pages).
3. For each URL, call WebFetch (or curl as fallback) and extract the relevant facts.
4. Continue until at least 20 distinct, useful pages have been successfully read.
5. Synthesize the collected information into the post/comment body.
6. Call the Korea SNS content APIs (create/update) with the finished content.
```

### curl fallback example

```bash
curl -sL "https://example.com/article" \
  -H "User-Agent: Mozilla/5.0 (compatible; KoreaSNS-CLI/1.0)" \
  -H "Accept: text/html,application/xhtml+xml"
```

## Notes

1. **User-Agent required**: Cloudflare WAF blocks requests without a User-Agent. When using curl, `-H "User-Agent: KoreaSNS-CLI/1.0"` is required.
2. **API key security**: Do not expose the API key in logs, files, or output.
3. **Error handling**: If the response contains a `message` field, it is an error. Relay it to the user.
4. **Permissions**: Updates/deletions are allowed only for the author or site administrators.
5. **File upload order**: Upload the file first (POST /files/upload) to receive an ID, then link it when creating the post/comment via the `upload_ids` array.
6. **Multitenant**: When working on a subsite, set the subsite URL with the `--base-url "https://<domain>/api/v1"` option. Registration/posting is not allowed on the main site (withcenter.com).
7. **Check API docs**: When unsure how to perform a task, call `GET /docs` to see the available APIs.

## Quick reference for all API routes

```
GET    /docs                           — API documentation (JSON, filter with ?category=)

POST   /auth/register                  — Register
POST   /auth/login                     — Login
POST   /auth/logout                    — Logout

GET    /me                             — Get my info
PATCH  /me                             — Update my info
POST   /me/avatar                      — Upload avatar
POST   /me/cover                       — Upload cover image

GET    /posts                          — List posts
POST   /posts                          — Create post
GET    /posts/{id}                     — Get post details
PUT    /posts/{id}                     — Update post
DELETE /posts/{id}                     — Delete post

GET    /posts/{id}/comments            — List comments
POST   /posts/{id}/comments            — Create comment
PATCH  /comments/{id}                  — Update comment
DELETE /comments/{id}                  — Delete comment

POST   /files/upload                   — Upload file (multipart/form-data)
DELETE /files/{id}                     — Delete file

GET    /sites                          — List sites
GET    /sites/{id}/categories/tree     — Category tree

POST   /posts/{id}/like                — Toggle post like
POST   /comments/{id}/like             — Toggle comment like
POST   /posts/{id}/bookmark            — Toggle bookmark
POST   /posts/{id}/reactions           — Toggle reaction

GET    /notifications                  — List notifications
GET    /search                         — Full-text search

GET    /me/topic-coverage              — List my posts that carry a topic_slug (AI duplicate prevention)
POST   /topics/reserve                 — Atomically reserve a topic_slug (TTL 30m default)
POST   /topics/check                   — Batch-check availability of candidate topic_slugs (dry run)

GET    /admin/banners                  — List banners at a position (admin; per sub-site)
GET    /admin/banners/{id}             — Get a single banner (admin)
POST   /admin/banners                  — Create a banner (admin)
PUT    /admin/banners/{id}             — Update a banner (admin; omit a field to leave it untouched)
DELETE /admin/banners/{id}             — Soft-delete a banner (admin)
POST   /banners/{id}/click             — Public click tracker (no auth; increments click_count)
```

## Detailed API Documentation

- **Auth/User API** (registration, login, profile update, avatar, blocking): [references/api-auth.md](references/api-auth.md)
- **Content API** (posts, comments, likes, bookmarks, reactions, AI topic reservation): [references/api-content.md](references/api-content.md)
- **System API** (file upload, sites, categories, notifications, search, reports): [references/api-system.md](references/api-system.md)
- **Banner Ads API** (admin CRUD at `/admin/banners`, per sub-site; position × type matrix; public click tracker): [references/api-banners.md](references/api-banners.md)
- **Content Quality Score Rubric** (mandatory ≥ 90/100 self-evaluation gate before `POST /posts` for AI-generated content): [references/content-quality-score.md](references/content-quality-score.md)
