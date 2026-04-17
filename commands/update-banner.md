---
name: update-banner
description: "Update a banner ad on a Korea SNS sub-site. Admin API key required. Example: '/korea:update-banner apple.withcenter.com 42 Change title to Summer Sale and extend end date to June 1'. Use for updating banners, editing banners, banner ads, extending banner schedule, changing banner image."
---

# /korea:update-banner — Update a Banner Ad

Update an existing banner. The endpoint uses `array_key_exists` semantics — only fields you pass are changed. Empty string on a nullable contact or date field clears it.

## Command Format

```
/korea:update-banner <sub-site domain> <banner_id> <prompt>
```

All three parts are required. `<banner_id>` must be numeric and must belong to the given sub-site (cross-site calls return 404).

## Usage Examples

```
/korea:update-banner apple.withcenter.com 42 Change title to "Summer Sale" and set end_at to 2026-06-01
/korea:update-banner bangphil.com 17 Clear the Telegram contact, set WhatsApp to +63 917 555 1234
/korea:update-banner cherry.withcenter.com 9 Swap the image to /tmp/new-banner.jpg
```

## Execution Procedure

### Step 1: Validate required parameters

| Information | Required | Description |
|-------------|----------|-------------|
| **Sub-site domain** | **O** | Admin's sub-site (e.g. `apple.withcenter.com`) |
| **Banner id** | **O** | Numeric id of the banner to update |
| **Prompt** | **O** | What to change |

**If anything is missing:**
```
Usage: /korea:update-banner <sub-site domain> <banner_id> <prompt>
Example: /korea:update-banner apple.withcenter.com 42 Change title to "Summer Sale" and end_at to 2026-06-01
```

### Step 2: Fetch the current banner

Always read before writing so the diff is grounded in reality:

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-show --id {BANNER_ID}
```

If the response is 404, the id is wrong or belongs to another sub-site — abort and tell the user.

### Step 3: Diff prompt vs current state

Compare the prompt against the current banner. Build a minimal flag set — **only pass flags for fields that actually change**. Never re-send a field with its current value.

Field ↔ flag mapping:

| Field to change | Flag |
|-----------------|------|
| `title` | `--title "…"` |
| `subtitle` | `--subtitle "…"` |
| `click_url` | `--click-url "…"` |
| `content` | `--content "…"` |
| `notes` | `--notes "…"` |
| `position` / `banner_type` / `category_id` | `--position`, `--banner-type`, `--category-id` |
| `begin_at`, `end_at` | `--begin-at "…"`, `--end-at "…"` (pass `""` to clear) |
| `between_interval`, `sort_order` | `--between-interval`, `--sort-order` |
| 7× `contact_*` | `--contact-telegram`, `--contact-phone`, …  (pass `""` to clear) |
| display image | `--upload-ids "new_id"` (replaces current image) |
| admin attachments | `--attachment-ids "id1,id2"` (appends) |
| advertiser attachments | `--advertiser-attachment-ids "id1,id2"` (appends) |

If the user wants to replace the image, first upload the new file (`upload --file …`) and use the returned id.

### Step 4: Perform the update

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-update --id {BANNER_ID} \
  --title "Summer Sale" --end-at "2026-06-01 00:00:00"
```

### Step 5: Report the result

Tell the user:
- Which fields were changed (before → after)
- Whether the banner is **currently active** after the update (dates bracket now)
- Any server warnings in the `message` field

Common errors: `404 Banner not found.` (wrong id or wrong sub-site), `403 Forbidden.` (admin key doesn't match this sub-site).

## Notes

- **Omit a field to leave it untouched.** Sending a current value as "just in case" is not a no-op — it overwrites.
- **Empty string clears nullable fields.** Works for `begin_at`, `end_at`, and the 7 `contact_*` fields.
- **`--upload-ids` replaces the image.** To keep it, omit the flag.
- **`--attachment-ids` / `--advertiser-attachment-ids` append.** To remove a file, delete it via `DELETE /files/{id}` rather than through banner-update.
- Full semantics: [../references/api-banners.md#3-put-adminbannersid--update-a-banner](../references/api-banners.md).
