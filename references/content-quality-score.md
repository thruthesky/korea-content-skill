# Content Quality Score System (100-point Rubric)

A self-evaluation rubric the AI **must apply to every generated post before calling `POST /posts`**. Posts scoring below **90** must not be published — the AI revises (up to 2 retries) or abandons the reservation.

The score system is the final quality gate between generation and submission. The `topic_slug` / reservation system in [api-content.md](api-content.md) prevents duplicate topics; this score system prevents **shallow, padded, or low-value posts** from polluting the site even when the topic is unique.

> 📌 **Scope**: This rubric is tuned for **place / venue / business guide** posts (e.g. restaurant, attraction, clinic, school, market guides for Korean expatriates). Such posts must include verifiable location data, contact info, and on-site photographs. Pure-text editorials are out of scope until a separate rubric is added.

---

## 1. When to apply the score

Apply the rubric at **Step 7** of the 7-step workflow (after claim extraction and the independent red-team pass, just before `POST /posts`):

```
Plan → Check → Reserve → Generate (research + draft + image upload)
                                                    ↓
                                         Extract claims (claims.json; §10)
                                                    ↓
                                   Red-team pass (external agent; §11) ── definite_errors? ──┐
                                                    ↓ clean                                   │ yes
                                            Score (self-eval; this rubric)                    ↓
                                                    ↓                                   revise or abandon
                               ┌────── pass (all 40 gates + ≥ 90/100) ──────┐
                               ↓                                             ↓
                    Submit (POST /posts)                              (fail → revise up to 2x)
```

- Apply the rubric **once per draft**, recording the subscore for every category.
- Score honestly. Inflating scores to get past the gate defeats the purpose.
- The red-team pass must return `definite_errors: []` — any definite error blocks submission even if the rubric score is high.
- If three drafts (1 original + 2 revisions) cannot clear all three bars (40 gates + ≥90 rubric + clean red-team), **release the reservation** (let it expire) and move to the next topic instead of publishing garbage.

### 1.1 Length tiers — target range by topic class

**New in v2 (replaces the old ≥5,000-word floor).** The old single floor forced every topic into exhaustive-guide territory, producing bloated posts. New policy: length follows topic depth with a **cap** so posts don't become encyclopedias.

| Topic class | Target Korean chars | Target English words | Typical read time | Examples |
|---|---|---|---|---|
| **Place / destination guide** | 4,500 – 7,000 | 2,500 – 4,000 | 8–12 min | Oslob whale shark, Boracay beaches |
| **How-to / visa / logistics** | 2,500 – 4,500 | 1,400 – 2,500 | 5–8 min | 9G visa 연장, GCash 가입, SIM 비교 |
| **Restaurant / single venue review** | 1,500 – 3,000 | 800 – 1,700 | 3–5 min | 세부 야키믹스 뷔페, BGC 한식당 |
| **Cultural deep-dive / long-form essay** | 5,000 – 8,000 | 2,800 – 4,500 | 10–15 min | 필리핀 가톨릭 문화, 필리피노 결혼식 |

G2 passes iff the Korean char count (or English word count) lands inside the band for the declared `topic_class`. If unsure which class applies, pick the one that best matches the topic's natural depth — don't force a restaurant review into a place-guide band.

---

## 2. Hard requirements (gates) — pass/fail before scoring

These must **all** be true before any rubric score is computed. If any item fails, the draft fails outright regardless of the rest of the rubric — do not submit; revise.

### 2.1 Content gates

| Gate | Requirement | How to verify |
|------|-------------|---------------|
| **G1**  | **≥ 20 distinct web sources** consulted during research | Working notes list 20+ unique domains |
| **G2**  | **Length within the target band for the topic class** — both undershooting (shallow) and overshooting (bloated) fail | Count Korean chars / words, compare against §1.1 tier |
| **G3**  | **≥ 5 top-level sections** (`##` headings) plus a final 📍 metadata section | Count `^##` lines in the Markdown source |
| **G4**  | Draft genuinely addresses the reserved `topic_slug` (no bait-and-switch) | Re-read intro and TOC; confirm topic match |
| **G5**  | `topic-check` confirmed the slug is `available` and not a trivial rephrase of an existing post | Re-run `topic-check` right before submit |
| **G6**  | Body is in the site's primary language (Korean for Korea SNS), coherent prose, not machine-translated | Read aloud; check honorific consistency |
| **G7**  | Body is written in **valid Markdown** using the **full syntax set** (see §2.4) | Render preview; verify all required syntax elements present |

### 2.2 Required structured metadata (REQUIRED — not optional)

Every post **must** include this metadata block at the very end of the body. Use the exact emoji + label format below so the data is parseable.

| Gate | Field | Format example |
|------|-------|----------------|
| **G8**  | 🏳️ **Country** | `Philippines` |
| **G9**  | 🗺️ **Region/Province/State** | `Cebu Province` |
| **G10** | 🏙️ **City** | `Cebu City` |
| **G11** | 📮 **Address** (full street address) | `123 Mango Avenue, Lahug, Cebu City 6000` |
| **G12** | 📌 **Coordinates (lat, lon)** — REQUIRED, ≥ 4 decimal places | `10.3157, 123.8854` |
| **G13** | 🏢 **Place / business / contact name** | `Sample Korean Restaurant` |
| **G14** | 📞 **Contact** (≥ 1 of: phone, email, KakaoTalk, Messenger, website) | `+63 32 123 4567 / kakao: sample` |

The metadata block goes inside a Markdown table at the end of the post (see §2.5).

**Optional** (do not gate, but include if known):
- 💰 **Pricing range**
- ⏰ **Opening hours**
- 🌐 **Website URL**
- 📱 **Social media handles**

If the AI cannot find verifiable values for the **required** fields G8–G14, **abandon the topic**. Do not invent addresses or coordinates — fabrication forces a 0 in §3.3 Accuracy.

### 2.3 Image gates

| Gate | Requirement | How to satisfy |
|------|-------------|----------------|
| **G15** | **≥ 3 images** embedded in the body | Three `![alt](url)` tags, none broken |
| **G16** | Every image is **downloaded locally first**, then **uploaded to the server** via `POST /files/upload`, then referenced by the **server-hosted URL** in the Markdown | Use the workflow in §4 |
| **G17** | Each image has a **descriptive `alt` text** (no `alt=""`, no generic "image1") | Lint each `![alt](url)`; alt ≥ 4 words |
| **G18** | At least one image shows the **place itself** (exterior, interior, or signage) | Visual review |
| **G19** | All `upload_ids` returned by `/files/upload` are **passed in `--upload-ids`** when calling `create`, AND the same URLs appear in the Markdown body | Cross-check upload IDs vs. body URLs |

> **Hotlinking external images is prohibited.** External image URLs go stale and the source site may not allow embedding. Always download → upload → embed via server URL.

### 2.4 Markdown syntax gates

The body must demonstrate **full Markdown literacy**. Each of these elements must appear at least once:

| Gate | Required Markdown element | Example |
|------|---------------------------|---------|
| **G20** | Headings of multiple levels — `##` and `###` (and `####` if structure warrants) | `## 개요`, `### 추천 메뉴` |
| **G21** | **Bullet lists** AND **numbered lists** (both must appear) | `- item` and `1. step` |
| **G22** | At least one **table** (besides the metadata table) | `| col | col |` |
| **G23** | **Bold** (`**text**`) AND *italic* (`*text*`) emphasis | `**중요**`, `*권장*` |
| **G24** | At least one **blockquote** (`>`) for quotes, tips, or warnings | `> 팁: 평일 방문 추천` |
| **G25** | At least one **inline link** with descriptive text — `[text](url)` (no bare URLs) | `[Google Maps에서 보기](https://maps.google.com/?q=10.3157,123.8854)` |
| **G26** | At least one **horizontal rule** (`---`) separating major sections | `---` |
| **G27** | At least one **inline code** (`` `code` ``) for prices/codes/identifiers, OR a **code block** (```) when relevant | `` ₱300 ``, fenced block for hours table |
| **G28** | At least one **image** uses Markdown syntax `![alt](url)` (counted toward G15 too) | `![식당 외관](https://...)` |

Bare HTML (e.g. `<table>`, `<div>`) is **not allowed** unless Markdown cannot express the construct. The site renders Markdown to HTML; do not pre-render.

### 2.5 Emoji / readability gates

| Gate | Requirement | Notes |
|------|-------------|-------|
| **G29** | **≥ 15 emojis or icons** spread across the body (intro, headings, key points, callouts) — not clustered in one section | Count any Unicode emoji or icon character |
| **G30** | **Every `##` heading** carries a leading emoji that matches the section topic | `## 🍜 추천 메뉴`, `## 🚗 가는 방법` |
| **G31** | **Average paragraph length ≤ 4 sentences**; no paragraph exceeds **6 sentences** | Walls of text fail this |
| **G32** | Lists, tables, and callouts appear at least **every 600 words** so the reader never sees an unbroken text block | Visual scan |

### 2.6 Anti-hallucination gates (NEW — G33–G38)

**Added in v2 after real incidents** (fabricated academic citation "Schleimer et al., 2018, *Travel and Tourism Ethics*"; nonsensical "1,300년 인류사" anchor; outdated emergency number 117 still published after PH switched to 911 in 2016; wrong embassy phone). These gates target the specific hallucination vectors that passed the old rubric.

| Gate | Requirement | How to verify |
|------|-------------|---------------|
| **G33** | **Every academic / journal citation** includes a DOI or a direct URL to the paper/abstract. A citation without one fails this gate — generic "Schleimer et al., 2018" without a link is treated as fabricated. | Every citation in the body maps to a real, reachable link. |
| **G34** | **Every phone number, emergency code, or contact hotline** links to the owning entity's **current** official page (embassy, municipal hotline, business website). Secondary blog mentions are not sufficient. | Each phone in claim ledger has `source_url` pointing to the owner's site. |
| **G35** | **Every law, statute, regulation, or local ordinance** cited (e.g. `Republic Act 10654`, `FAO No. 193`, `Badian Municipal Ordinance No. 11-2022`) links to the official text or — for local ordinances that aren't indexed online — to a reputable news article confirming both the ordinance **number** and **year**. Cited law/ordinance must actually cover the claimed subject matter. **Local ordinances** are the hardest subcategory: fabricating an ordinance number with a plausible-sounding year is a common LLM failure mode. If a specific ordinance number can't be verified, soften to "the local government has standardized the rate" instead of citing a fake number. | Follow each cited number; if the source is a news article (not a primary text), the article must explicitly name both the number and year. |
| **G36** | **Every "since Year" / "N-year history" / specific anchor-date claim** has a source. No inventing anchor durations. `"1,300년 인류사"` without a source is a fabrication. | Each such phrase has a `source_url` in the claim ledger. |
| **G37** | **Every absolute claim** — "no recorded cases", "only operation in the world", "largest ever", "first X in Philippines" — has a sourced basis. Vague absolutes ("few documented") are preferred when a hard absolute can't be sourced. | Scan the draft for absolute language; each one has a source. |
| **G38** | **No high-risk-class claim** (`safety`/`legal`/`financial`/`contact`) ships with `confidence < "high"` in the claims.json. Low-confidence high-risk claims are either dropped or replaced with a verified value — not published. | Cross-check risk_class × confidence in claims.json. |

**If any of G33–G38 fails, the draft is revised or the affected claim is removed. Publishing a fabricated citation or wrong emergency number is a first-order credibility failure — worse than a missing metadata field.**

### 2.7 Extended fact-verification gates (NEW — G39–G40)

**Added in v2.1 after a second round of real incidents** (Kawasan Falls / canyoneering post: fabricated administrative classification *"Badian Autonomous City"* — PH has no such classification; misspelled proper nouns *"Pagsanhan"* (should be Pagsanjan) and *"En Salada"* (should be Ensalada); unsourced *"40m waterfall"* when the main tier is actually ~30m; invented geographic features like *"Kabukalan Spring"* and *"Sardine Point"*; an activity ↔ location mismatch *"Davao bungee jumping"* when the PH bungee is at Danao Adventure Park in Bohol). These gates close coverage gaps that G33–G38 didn't explicitly address.

| Gate | Requirement | How to verify |
|------|-------------|---------------|
| **G39** | **Every proper noun** — place names, administrative classifications (city / municipality / barangay / province / region / autonomous region), dish or local-food names, foreign-loan words, geographic features (springs, rivers, headwaters, beaches, peaks), organization names — appearing in any **heading**, **table cell**, **📍 metadata block**, or **3+ times** in the body has a `source_url` in the claim ledger. **Spelling and classification must match the cited source verbatim.** Invented classifications like "Autonomous City" (outside BARMM) or invented feature names like "Kabukalan Spring" (no such documented source) fail this gate. | Every qualifying proper noun has an entry; red-team cross-checks spelling and administrative type against Wikipedia / PSA / PhilAtlas / official LGU records. |
| **G40** | **Every numeric physical measurement** (height, depth, distance, area, weight, duration, capacity, elevation, slope, flow rate) has a `source_url`. Rounding is acceptable when labeled ("약 30m", "~30m", "approximately 30m"); an unlabeled specific like "40m" without a sourced value is treated as a fabrication. **For measurements from multiple sources that disagree (e.g., 30m vs. 40m),** pick the one with the most authoritative source or use a labeled range ("약 30~40m"). | Each measurement in the claim ledger has a source; red-team cross-checks against authoritative data (Wikipedia, gov tourism page, topographic source). |

**If G39 or G40 fails, revise by replacing unverified specifics with sourced ones, softening to labeled approximations, or abstracting to a common-noun alternative.** A common-noun fallback for G39 is often the cleanest fix — *"a three-tiered limestone waterfall"* is sourced-by-default, whereas *"a three-tiered 40-meter waterfall fed by Kabukalan Spring"* piles specifics that each need verification.

---

### Required metadata block template (paste at end of body, fill in real values)

```markdown
---

## 📍 방문 정보 (Place Info)

| 항목 | 내용 |
|------|------|
| 🏳️ 국가 (Country) | Philippines |
| 🗺️ 지역 (Region) | Cebu Province |
| 🏙️ 도시 (City) | Cebu City |
| 📮 주소 (Address) | 123 Mango Avenue, Lahug, Cebu City 6000 |
| 📌 좌표 (Lat, Lon) | 10.3157, 123.8854 — [Google Maps](https://www.google.com/maps?q=10.3157,123.8854) |
| 🏢 이름 (Name) | Sample Korean Restaurant |
| 📞 연락처 (Contact) | +63 32 123 4567 / kakao: `sample` / sample@example.com |
| 💰 가격대 (Pricing, optional) | ₱300 ~ ₱1,500 / 인 |
| ⏰ 영업시간 (Hours, optional) | 매일 10:00 ~ 22:00 (수요일 휴무) |
| 🌐 웹사이트 (optional) | https://example.com |
```

If a gate fails, stop and fix **that specific problem** before scoring the rubric.

---

## 3. Rubric — 100 points across 6 categories

The 40 gates above guarantee **structural minimums** and **freedom from the known hallucination vectors**. The 6 categories below evaluate **quality**. Even a draft that passes all gates can score 50/100 if it is hollow — gate-pass is necessary but not sufficient.

### 3.1 Originality & Uniqueness — 20 points

Does this post bring something new to the site, or is it a reworded rehash of easily-Googled content?

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 17–20 | Fresh framing, original analysis, comparisons across multiple visits/sources, or first-hand observations not found in the top 10 search results. Includes Korean-expat-specific angles (visa/insurance/Hangul signage notes). |
| Good | 13–16 | Solid synthesis of multiple sources. Some original commentary or sequencing, but no insight a reader couldn't assemble themselves. |
| Fair | 9–12 | Mostly a reorganization of existing material. Few original sentences. |
| Weak | 0–8 | Closely paraphrases a single source (e.g. one TripAdvisor page). Reader could find this trivially elsewhere. |

**Red flags (force ≤ 8):**
- Opening paragraph reads like Wikipedia's first sentence
- Content follows the structure of the #1 search result line-by-line
- No original examples, comparisons, or observations

### 3.2 Depth & Substance — 20 points

Does the post go beyond surface-level definitions into useful detail?

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 17–20 | Concrete examples, numbers, dates, comparisons, trade-offs. Step-by-step "가는 방법", menu/service breakdown, peak-hour analysis. Reader leaves ready to act. |
| Good | 13–16 | Explains the "why" as well as the "what". Several concrete examples. |
| Fair | 9–12 | Covers the "what" adequately but thin on "why" and "how". |
| Weak | 0–8 | Bullet-point definitions only. Padding phrases ("In today's world…"). 5,000 words but 4,000 of them are filler. |

**Red flags (force ≤ 8):**
- No specific numbers (prices, distances, durations, capacities)
- No comparisons (between options, regions, time periods, alternatives)
- Filler sentences that say nothing ("This is a very important place to visit.")

### 3.3 Accuracy & Verifiability — 15 points

Are factual claims correct and traceable to credible sources? **Coordinates, address, and contact info must be verifiable.**

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 13–15 | Every non-obvious claim grounded in ≥ 1 credible source from the working notes. Coordinates verified against Google Maps / OpenStreetMap. Phone numbers verified against official sources. Numbers current within 12 months for volatile data (prices, hours). |
| Good | 9–12 | Most claims supported. Coordinates and address correct. Minor outdated references. |
| Fair | 5–8 | Some unverifiable claims. Approximated coordinates (only 2-3 decimals). Stale prices. |
| Weak | 0–4 | Invented names, wrong dates, hallucinated coordinates, non-existent phone numbers. |

**Red flags (force 0):**
- Any fabricated address, coordinate pair, phone number, or URL
- Coordinates that don't match the stated city when checked
- Any legal/medical/financial claim without a credible source

### 3.4 Structure, Markdown & Readability — 15 points

Is the post easy to scan, navigate, and visually pleasant? Does it use the full Markdown toolset effectively?

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 13–15 | Clear title. ≥ 5 well-labeled sections, all with topic-matching emoji headings. Multiple tables, lists, blockquotes used **where they help comprehension**. Short paragraphs. Smooth visual rhythm — text/list/image/table alternation. Full Markdown syntax used naturally. |
| Good | 9–12 | Logical sections. Short paragraphs. Uses most Markdown elements but a few feel forced. One or two walls of text. |
| Fair | 5–8 | Sections exist but headings are vague. Many long paragraphs. Markdown syntax used minimally — meets gates but no flair. |
| Weak | 0–4 | Single wall of text. Vague headings ("정보", "기타"). Tables/lists/emojis missing or clustered. Reads like a doc dump. |

**Red flags:**
- A single paragraph > 6 sentences (−2)
- No table or list in a 5,000+ word post (−3)
- Heading text identical to the post title (−1)
- All emojis crammed into one section, none in others (−2)

### 3.5 Reader Value — 20 points

Will the target audience (Korean expatriates) actually benefit?

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 17–20 | Tailored to Korean expat context: KRW conversions for prices, visa/SIM/transport notes, language tips (does the staff speak Korean/English?), nearby Korean community resources, "이런 분께 추천" persona guidance. Includes warnings, alternatives, and "how to verify this yourself" pointers. |
| Good | 13–16 | Generally useful. Mentions local context but misses one or two opportunities. |
| Fair | 9–12 | Generic global advice. No local specifics. Reads like it could be about anywhere. |
| Weak | 0–8 | Off-topic, irrelevant, or so generic it adds no value over a random blog. |

**Red flags (force ≤ 8):**
- All prices in USD only with no KRW or local-currency equivalent
- No mention of any Korea- or expat-specific nuance
- Map link missing despite coordinates being present (always add `https://www.google.com/maps?q={lat},{lon}`)

### 3.6 Polish — 10 points

Final presentation quality.

| Band | Score | Description |
|------|------:|-------------|
| Excellent | 9–10 | Zero typos. Consistent honorific level. Proper Korean spacing/particles. Emojis natural and on-topic. Markdown renders cleanly with no broken syntax. No machine-translation artifacts. |
| Good | 7–8 | 1–2 minor typos or awkward sentences. Markdown all valid. |
| Fair | 4–6 | Multiple typos. Noticeably stilted. Inconsistent tone. Some Markdown rendering glitches. |
| Weak | 0–3 | Broken grammar, garbled sentences, untranslated English fragments inside Korean body, broken `![alt(url)` syntax. |

**Red flags (force 0):**
- Any `[TODO]`, `[FIXME]`, lorem ipsum, or placeholder text
- Any obvious translation artifact (e.g. "It is a kind of place" for 곳입니다)
- A broken image (404 when fetched)

---

## 4. Image workflow (mandatory before submit)

Hotlinking external images is forbidden (see G16). The flow is **download → upload → embed**.

```bash
# 1) Download promising images during research. Use curl with a real UA.
mkdir -p /tmp/post-images
curl -L -o /tmp/post-images/exterior.jpg \
  -H "User-Agent: Mozilla/5.0" \
  "https://example.com/path/to/photo.jpg"

# 2) Upload each image to the Korea SNS server. The response gives an upload id and a server URL.
python3 .claude/skills/content/scripts/korea_api.py \
  --api-key "{KEY}" --base-url "{BASE}" \
  upload --file /tmp/post-images/exterior.jpg
# Example response:
# {"data": {"id": 124, "url": "https://withcenter.com/uploads/2026/04/abc.jpg"}}

# 3) Repeat for at least 3 images. Collect all upload IDs and URLs.

# 4) Embed each image inside the Markdown body using the server URL:
#    ![식당 외관 — 망고 애비뉴 입구](https://withcenter.com/uploads/2026/04/abc.jpg)

# 5) When calling create, pass the IDs in --upload-ids:
python3 .claude/skills/content/scripts/korea_api.py \
  --api-key "{KEY}" --base-url "{BASE}" \
  create --title "..." --content "$BODY_MARKDOWN" \
  --topic-slug "ph-cebu-sample-restaurant" --reservation-id 1 \
  --upload-ids "124,125,126"
```

**Rules**:
- Image alt text describes the **subject**, not the file. `![식당 외관]` ✅, `![image1]` ❌.
- Place at least one image **near the top** (after the intro) and spread the rest through the body — do not dump them all in a single gallery section.
- Verify the rendered URL works **before** scoring (it must be the server URL, not the upload `id` or local path).
- Respect copyright — prefer your own photos, Wikimedia Commons, or sources that explicitly allow embedding.

---

## 5. Self-evaluation output

The AI must produce a JSON block like this **internally** (not posted publicly) before deciding to submit:

```json
{
  "topic_slug": "ph-cebu-sample-restaurant",
  "word_count": 5421,
  "section_count": 9,
  "source_count": 23,
  "image_count": 5,
  "metadata": {
    "country": "Philippines",
    "region": "Cebu Province",
    "city": "Cebu City",
    "address": "123 Mango Avenue, Lahug, Cebu City 6000",
    "lat": 10.3157,
    "lon": 123.8854,
    "name": "Sample Korean Restaurant",
    "contact": "+63 32 123 4567 / kakao: sample"
  },
  "markdown_features": ["h2","h3","ul","ol","table","bold","italic","blockquote","link","hr","inline-code","image"],
  "emoji_count": 32,
  "gates_passed": ["G1","G2","G3","G4","G5","G6","G7","G8","G9","G10","G11","G12","G13","G14",
                   "G15","G16","G17","G18","G19","G20","G21","G22","G23","G24","G25","G26",
                   "G27","G28","G29","G30","G31","G32","G33","G34","G35","G36","G37","G38",
                   "G39","G40"],
  "scores": {
    "originality": 18,
    "depth": 17,
    "accuracy": 14,
    "structure": 13,
    "reader_value": 19,
    "polish": 9
  },
  "total": 90,
  "pass": true,
  "weak_areas": [],
  "revision_round": 0
}
```

**Required fields:**
- All 40 gates must appear in `gates_passed`. Any missing gate → fail outright, do not submit.
- All 6 scores must be present (no `null`).
- `total` is the sum of all 6 scores (max 100).
- `pass = total ≥ 90 AND len(gates_passed) == 40 AND red_team.definite_errors == []`.
- `weak_areas`: every category scoring below the "Good" band (`originality < 13`, `depth < 13`, `accuracy < 9`, `structure < 9`, `reader_value < 13`, `polish < 7`).
- `revision_round`: 0 for first draft, 1 for first revision, 2 for second revision.

---

## 6. Revision loop (when `pass = false`)

```
if (gate_failed or total < 90) and revision_round < 2:
    fix_gate_failures()           # gates first — they are absolute
    rewrite_weak_areas()          # then targeted quality edits
    revision_round += 1
    re-score
else:
    abandon_topic()               # do NOT call POST /posts
    # The reservation will auto-expire after TTL; no explicit release endpoint is needed.
    pick_next_topic()
```

### Targeted fixes per failure

| Failure | Fix |
|---------|-----|
| **G2 length band** | **Undershoot**: add depth-driven sections (history, comparison with alternatives, persona-targeted advice, FAQ). Do NOT pad with filler — that tanks §3.2. **Overshoot**: tighten by merging overlapping sections, removing tangential content, and cutting redundant adjectives. Consider demoting to a smaller topic class (e.g., a single-venue review rather than a destination guide) if the material genuinely doesn't warrant the higher tier. Both undershoot and overshoot fail G2 equally. |
| **G33–G38 anti-hallucination** | Extract every flagged claim into `claims.json` with `source_url` + `confidence` + `risk_class` (§10). For `definite_errors` raised by the red-team (§11), either remove the specific, soften to a sourced vague alternative, or research and link a primary source. High-risk classes (safety / legal / financial / contact) must be sourced or dropped — never softened. |
| **G39 proper nouns** | For each failing proper noun: (a) look up the authoritative spelling on Wikipedia / PhilAtlas / PSA / the owning LGU's page and correct it; (b) verify the administrative classification (e.g., "Municipality of Badian" not "Badian Autonomous City"); or (c) replace the named specific with a common-noun alternative ("a natural spring" instead of a fabricated named spring). |
| **G40 measurements** | For each failing measurement: (a) source a number from Wikipedia / government tourism data / a topographic source and add `source_url` to the claim ledger; (b) convert the specific number to a labeled approximation ("~30m", "약 30m"); or (c) drop the specific altogether ("a three-tiered waterfall" instead of "a 40-meter three-tiered waterfall"). |
| **G15/G16 image** | Source 1–2 more images from a different domain. Confirm download → upload → embed pipeline. Verify `--upload-ids` flag is passed. |
| **G8–G14 metadata** | Re-research; if any field can't be verified, abandon the topic — do not invent. |
| **G12 coordinates** | Use Google Maps or OpenStreetMap to obtain coordinates ≥ 4 decimals. Round at 4 decimals only if higher precision is unavailable. |
| **G20–G28 Markdown syntax** | Pass through the body; convert at least one paragraph to a list, one comparison to a table, one tip to a blockquote, one URL to a labeled link. |
| **G29–G30 emoji** | Add one topic-matching emoji per `##` heading; sprinkle 1–2 emojis per major bullet list. |
| **G31/G32 readability** | Split long paragraphs at every clause break ≥ 4 sentences. Insert subheadings, lists, or pull-quotes. |
| **Originality low** | Add a contrarian angle, a comparison no one else has done, or a personal/case observation. Remove paragraphs that mirror the #1 Google result. |
| **Depth low** | Insert specific numbers, comparisons, and trade-offs. Convert generic prose into tables. Remove filler sentences. |
| **Accuracy low** | Cross-check every fact against ≥ 2 sources. Remove unsupported assertions. Refresh stale figures. Re-verify coordinates. |
| **Structure low** | Split long paragraphs. Add sub-headings. Add comparison tables. Use blockquotes for tips. |
| **Reader value low** | Add a "한국인 방문 시 팁" subsection. Add KRW conversions. Add Google Maps link. Add transport directions. |
| **Polish low** | Re-read aloud. Fix typos, awkward particles. Validate every Markdown link by mental rendering. |

**Rules for revisions:**
1. Round 1: targeted edits only — keep the parts that scored well.
2. Round 2 may rewrite larger sections but must preserve the reserved `topic_slug` (the reservation is for that specific slug).
3. Do not re-use any sentence the previous round flagged as filler or inaccurate.

---

## 7. Worked example

**Topic reserved:** `ph-cebu-yakimix-buffet`
**Draft 1 result:**
```json
{
  "word_count": 3200,
  "image_count": 1,
  "metadata": { "lat": null, "lon": null, ... },
  "gates_passed": ["G1","G3","G4","G5","G6","G7","G8","G9","G10","G11","G13","G14",
                   "G17","G18","G20","G21","G23","G25","G26","G29","G31"],
  "missing_gates": ["G2","G12","G15","G16","G19","G22","G24","G27","G28","G30","G32"],
  "scores": { "originality":11,"depth":10,"accuracy":12,"structure":13,"reader_value":12,"polish":8 },
  "total": 66,
  "pass": false,
  "revision_round": 0
}
```

**Round 1 targeted edits:**
- G2: Expanded from 3,200 → 5,400 words by adding "메뉴 카테고리별 분석", "한국인 방문 팁 5가지", "주변 관광지", "FAQ" sections — all with substance, not filler.
- G12: Looked up `10.3219, 123.9050` on Google Maps; confirmed it matches Lahug Cebu City; added to metadata.
- G15/G16/G19: Downloaded 4 more photos (interior, sushi bar, dessert station, exit), uploaded each via `/files/upload`, embedded via `![alt](server-url)`, passed all 5 IDs to `--upload-ids`.
- G22: Added a comparison table of 5 popular dishes (price / portion / Korean palate match).
- G24: Added a `> 💡 팁: 평일 저녁 7시 이후가 한국인 가족 단위에 가장 한산` blockquote.
- G27: Wrapped prices like `` ₱599 ``, `` ₱799 `` in inline code.
- G28: Confirmed Markdown image syntax (was using HTML before).
- G30: Added emoji to every `##` heading: `## 🍣 메뉴 구성`, `## 🚗 가는 방법`, etc.
- G32: Inserted callouts and lists every ~500 words.
- Originality: Added Yakimix vs. Vikings vs. Buffet 101 comparison.
- Depth: Added per-station observations + price-per-person analysis.
- Reader value: Added KRW conversions and "한국인 가족 방문 시 추천 시간대".

**Draft 2 result:**
```json
{
  "word_count": 5421, "image_count": 5,
  "gates_passed": [/* all 40 */],
  "scores": { "originality":17,"depth":18,"accuracy":14,"structure":14,"reader_value":19,"polish":9 },
  "total": 91,
  "pass": true
}
```
→ Submit.

---

## 10. Claim extraction & source attribution (NEW — step 5 of the 7-step workflow)

Before scoring, the AI **must** extract every testable factual claim from the draft into a structured JSON file (`claims.json`). This step converts a prose draft into a machine-auditable inventory of assertions — the critical step that exposes hallucinations that prose-level review misses.

### Why this step exists

A model grading its own post cannot detect its own hallucinations — it wrote them because it believed (incorrectly) they were true. Extracting claims into a flat list **before** scoring forces the model to confront each assertion individually: "what URL supports this?" Any claim without a source is either dropped, softened, or researched.

### What counts as a "testable claim"

Every specific factual assertion:

- Numbers (prices, distances, dates, durations, capacities, percentages, char/word counts)
- Proper names (people, organizations, products, places, laws)
- Phone numbers, addresses, coordinates, emails, URLs
- Quoted statements and statistics from research
- Specific law or regulation references (`Republic Act 10654`, `FAO No. 193`, etc.)
- Academic citations (author, year, journal)
- Absolute claims ("no recorded cases", "only one in the world", "first in Philippines")
- Anchor-dates ("since 2011", "over 1,300 years of")
- Category-defining claims ("the largest fish", "IUCN Endangered status")

Narrative prose and subjective evaluations ("aesthetically pleasing", "highly recommended") do NOT need entries — only **testable** facts.

### Schema

```json
{
  "topic_slug": "ph-cebu-oslob-whale-sharks",
  "draft_char_count_korean": 6200,
  "extracted_at": "2026-04-22T02:29:00Z",
  "claims": [
    {
      "id": "C001",
      "quote": "필리핀 경찰 긴급번호 911",
      "claim": "PH national emergency number is 911",
      "source_url": "https://www.doilg.gov.ph/news/national-emergency-hotline-911",
      "source_quote": "Republic Act 10844 consolidated all emergency hotlines under 911, effective August 2016.",
      "date_accessed": "2026-04-22",
      "confidence": "high",
      "risk_class": "safety"
    },
    {
      "id": "C002",
      "quote": "Schleimer et al., 2018 (Travel and Tourism Ethics)",
      "claim": "A 2018 paper in a journal called 'Travel and Tourism Ethics' by Schleimer et al. reported Guilty-Pleasure tourist behavior",
      "source_url": null,
      "confidence": "none",
      "risk_class": "descriptive",
      "action": "REMOVE — no journal named 'Travel and Tourism Ethics' exists; citation fabricated"
    }
  ],
  "stats": {
    "total": 47,
    "sourced_high": 38,
    "sourced_medium": 5,
    "sourced_low": 2,
    "no_source_action_remove": 2,
    "risk_class_counts": { "safety": 4, "legal": 3, "financial": 6, "contact": 5, "descriptive": 29 }
  }
}
```

### Required fields per claim

- **`id`**: Sequential identifier for cross-reference.
- **`quote`**: Exact text from the draft (short fragment).
- **`claim`**: The factual assertion being made, paraphrased concisely.
- **`source_url`**: URL to a credible source. `null` is allowed only if action is REMOVE or SOFTEN.
- **`source_quote`**: Exact text from the source supporting the claim (≤ 200 chars). Required when `source_url` is set.
- **`date_accessed`**: YYYY-MM-DD when the source was fetched.
- **`confidence`**: `high` (primary source, recent) / `medium` (secondary source or older) / `low` (uncertain) / `none` (unsourced).
- **`risk_class`** (one of):
  - `safety` — emergency numbers, embassy contacts, medical info, weather warnings, legal compliance
  - `legal` — law citations, regulation numbers, visa rules, taxation
  - `financial` — prices, fees, currency rates, investment info
  - `contact` — phone numbers, addresses, emails, hours, coordinates
  - `descriptive` — history, culture, biology, opinion, flavor notes — anything not in the above four
- **`action`** (optional, when claim doesn't meet quality bar): `KEEP` / `REMOVE` / `SOFTEN` / `RESEARCH`.

### Pass/fail rules

The claims.json step passes iff:

1. Every claim has `source_url` OR an `action` of `REMOVE`/`SOFTEN` that has been applied to the draft.
2. Every claim with `risk_class` in `{safety, legal, financial, contact}` has `confidence: "high"`. Lower-confidence high-risk claims must be dropped or verified.
3. Every `action: REMOVE` claim has actually been removed from the draft.
4. Every `action: SOFTEN` claim has been rephrased to a sourced vague alternative ("매년 수만 명" instead of "연간 45,000명" when the latter is unsourced).
5. The draft's claim density aligns with `stats.total` — if the draft contains more facts than are in the ledger, the ledger is incomplete.

If any rule fails, revise and re-extract. Do not skip this step to save time — this is the single step that catches fabricated citations and invented numbers.

---

## 11. Red-team / external fact-check pass (NEW — step 6 of the 7-step workflow)

After claims.json is clean, spawn an **independent fact-check agent** (fresh context, ideally a different model invocation) and hand it the draft + claim ledger. The agent's job is to red-team the post with no loyalty to the author.

### Why this step exists

Steps 5 (claim extraction) and 7 (self-score) are both performed by the authoring model. Even when the claim ledger is diligently built, the model can still miss its own fabrications because it *believes* them to be true — the same reason they were hallucinated in the first place. An independent agent with fresh context catches what the author cannot see.

### Red-team prompt template

Use this (or equivalent) as the prompt for the external agent:

```
You are fact-checking a Korean-language article drafted by another AI for a Korean-expat
community site. Your job is to find factual errors — fabrications, outdated info, wrong
numbers, misattributed citations, wrong statutes, fake phone numbers. Do NOT evaluate style,
structure, or word choice. Only facts.

Article (Markdown):
<<<
{FULL_DRAFT}
>>>

Claim ledger (claims.json):
<<<
{CLAIMS_JSON}
>>>

Return strictly valid JSON with three arrays — no prose commentary:

{
  "definite_errors": [
    { "quote": "exact text from draft",
      "problem": "what's wrong",
      "evidence": "what the truth is and how you verified",
      "severity": "safety|legal|financial|contact|credibility"}
  ],
  "likely_errors": [
    { "quote": "...", "problem": "...", "evidence": "..." }
  ],
  "plausible_unverified": [
    { "quote": "...", "note": "plausible but not verified — user should check" }
  ]
}

Rules:
- A claim is a "definite error" only if you can cite a contradicting primary source
  (official site, published paper, government page). Otherwise it's "likely" or "plausible".
- Every specific phone number, coordinate, and law must be verified against the owning
  entity's own website — do not rely on blog summaries.
- If a cited paper / journal has no findable DOI or URL, that's a definite error (fabrication).
- Fix suggestions are NOT your job. Just identify problems.
```

### Pass/fail rules

- If `definite_errors.length > 0` → **block submission**. Revise the draft to address each one, re-extract claims, re-run red-team. Max 2 revision rounds; if still failing, abandon the reservation.
- If `likely_errors.length > 0` → revise or remove each, unless the author can provide a primary source that contradicts the red-team's assessment.
- `plausible_unverified` is informational — included in the final review log but does not block submission.

### Model diversity (optional but recommended)

The strongest red-team is a different model family than the author. If the author is Claude Opus, use Claude Sonnet or Haiku for the red-team; different training data distributions catch different blindspots. If only one model is available, a fresh agent instance with the prompt above still helps because it has no exposure to the draft's generation context.

---

## 12. Verification by risk class (NEW)

Not every claim needs the same verification rigor. High-risk claims (safety / legal / financial / contact) must be verified against primary sources — the owning entity's own page. Descriptive prose can lean on secondary sources aggregated during the ≥20-source research phase.

| Risk class | What requires verification | Acceptable sources | Unacceptable sources |
|---|---|---|---|
| **safety** — emergency numbers, embassy contacts, medical, weather | Every specific number/address must be on the owning government agency / embassy / hotline site, verified within 12 months | `.gov.ph`, `.mofa.go.kr`, WHO, PH Red Cross, official embassy site | Travel blogs, forum posts, Wikipedia (for current contact info) |
| **legal** — law citations, visa rules, regulation numbers | Every statute number must link to official text; the cited law must actually cover the claim | `lawphil.net`, `officialgazette.gov.ph`, `senate.gov.ph`, BI, BIR, DFA | Secondary blog summaries, Stack Exchange |
| **financial** — prices, fees, rates | Must be dated within 12 months of publication, ideally from the vendor's own current rate card or a recent (≤3 months) news report | Official tariff pages, vendor website, recent news | Year-old blog posts, outdated tour packagers |
| **contact** — phones, addresses, coordinates | Phone/address must come from the owning business's own current page; coordinates must be verified on Google Maps / OpenStreetMap | Business's own site, Google Maps pin | Third-party aggregators that may lag |
| **descriptive** — history, culture, biology, economics analysis | Needs ≥ 1 credible secondary source from the research phase; 2+ for contested claims | News outlets, academic papers, encyclopedias, research institutes | Unsourced claims, forum opinions alone |

Every claim in `claims.json` must pass the verification bar for its `risk_class`. Apply this at step 5 (claim extraction) — don't defer to the red-team.

---

## 8. Interaction with topic reservation

- The rubric runs **after** a successful `POST /topics/reserve`. The reservation holds the slug for the TTL (default 30 minutes). Use this window for image download/upload, drafting, scoring, and up to 2 revisions.
- Long-form posts (cultural deep-dives or full place guides in the 5,000–8,000 Korean-char tiers, plus 3+ images) may exceed the default TTL. Pass `--ttl-minutes 90` (max 120) when reserving these. Short-form reviews (1,500–3,000 chars) typically fit inside the default 30-minute window.
- If scoring exhausts the TTL, the reservation expires naturally; the next `topic-reserve` call for the same slug will succeed again (expired rows are lazy-cleaned). However, your uploaded images will already be on the server — reuse those upload_ids if you re-reserve the same slug.
- **Do not** call `POST /posts` with `reservation_id` until every gate passes and the rubric returns `pass: true`. Submitting a low-quality post consumes the reservation and permanently locks the slug against re-publishing for this user — wasting a reservation on junk is worse than abandoning it.

---

## 9. Quick checklist (copy this into your drafting workspace)

```
=== Hard requirements ===
[ ] G1   ≥ 20 distinct sources consulted
[ ] G2   Length inside target band for declared topic class (§1.1)
[ ] G3   ≥ 5 sections + 📍 metadata block
[ ] G4   Body matches reserved topic_slug
[ ] G5   topic-check returned "available"
[ ] G6   Language coherent and target-language
[ ] G7   Body is valid Markdown using full syntax

=== Metadata (G8–G14) ===
[ ] G8   🏳️ Country
[ ] G9   🗺️ Region/Province
[ ] G10  🏙️ City
[ ] G11  📮 Address (full)
[ ] G12  📌 Lat, Lon (≥ 4 decimals) + Google Maps link
[ ] G13  🏢 Place / business / contact name
[ ] G14  📞 Contact (phone / email / Kakao / etc.)

=== Images (G15–G19) ===
[ ] G15  ≥ 3 images embedded
[ ] G16  All images downloaded → uploaded → server URL embedded
[ ] G17  Descriptive alt text on every image
[ ] G18  ≥ 1 photo of the place itself
[ ] G19  --upload-ids passed and matches body URLs

=== Markdown syntax (G20–G28) ===
[ ] G20  ## and ### headings
[ ] G21  Bullet list AND numbered list
[ ] G22  Non-metadata table
[ ] G23  **bold** and *italic*
[ ] G24  Blockquote (>)
[ ] G25  Inline link [text](url)
[ ] G26  Horizontal rule (---)
[ ] G27  Inline code or code block
[ ] G28  Markdown image syntax ![alt](url)

=== Emoji & readability (G29–G32) ===
[ ] G29  ≥ 15 emojis spread across body
[ ] G30  Every ## heading has a leading topic emoji
[ ] G31  No paragraph > 6 sentences, average ≤ 4
[ ] G32  Lists/tables/callouts every ~600 words

=== Anti-hallucination (G33–G38) ===
[ ] G33  Every academic citation has a DOI or direct URL
[ ] G34  Every phone / emergency number linked to owning entity's official page
[ ] G35  Every law / regulation / local-ordinance number linked to official text (or news article naming both number + year)
[ ] G36  Every "since Year" / anchor-date claim sourced
[ ] G37  Every absolute claim ("no recorded cases" etc.) sourced
[ ] G38  No safety/legal/financial/contact claim with confidence < "high"

=== Extended fact-verification (G39–G40) ===
[ ] G39  Every proper noun in heading/table/📍 metadata (or used 3+ times) has source_url with matching spelling + admin type
[ ] G40  Every physical measurement (height, depth, distance, duration, capacity) has source_url or is a labeled approximation

=== Claim ledger (step 5) ===
[ ] claims.json produced with every testable assertion
[ ] Every claim has source_url OR is REMOVEd/SOFTENed
[ ] Every safety/legal/financial/contact claim has confidence "high"
[ ] Every REMOVE action actually applied to the draft

=== Red-team pass (step 6) ===
[ ] Independent fact-check agent run with draft + claims.json
[ ] definite_errors: [] (empty — any entry blocks submission)
[ ] likely_errors addressed or countered with primary source

=== Scores (must sum ≥ 90) ===
Originality   /20 _____
Depth         /20 _____
Accuracy      /15 _____
Structure     /15 _____
Reader Value  /20 _____
Polish        /10 _____
Total         /100 _____

If any gate fails → fix that first.
If total < 90 → revise weak areas (max 2 rounds) or abandon the reservation.
```
