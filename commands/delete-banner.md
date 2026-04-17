---
name: delete-banner
description: "Soft-delete a banner ad on a Korea SNS sub-site. Admin API key required. Example: '/korea:delete-banner apple.withcenter.com 42'. Use for deleting banners, removing banner ads, taking a banner offline."
---

# /korea:delete-banner — Delete a Banner Ad (soft)

Soft-deletes a banner (sets `deleted_at`). The row is excluded from every active-render and admin-list query. There is no API to restore.

## Command Format

```
/korea:delete-banner <sub-site domain> <banner_id>
```

Both are required.

## Usage Examples

```
/korea:delete-banner apple.withcenter.com 42
/korea:delete-banner bangphil.com 17
```

## Execution Procedure

### Step 1: Validate required parameters

| Information | Required | Description |
|-------------|----------|-------------|
| **Sub-site domain** | **O** | Admin's sub-site (e.g. `apple.withcenter.com`) |
| **Banner id** | **O** | Numeric id of the banner to delete |

**If missing:**
```
Usage: /korea:delete-banner <sub-site domain> <banner_id>
Example: /korea:delete-banner apple.withcenter.com 42
```

### Step 2: Show the target and confirm

Deletion is only recoverable via direct DB access, so always show the caller what they're about to remove before running the delete:

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-show --id {BANNER_ID}
```

Summarize for the user: title, position/type, `begin_at`–`end_at`, current active status. Ask them to confirm (`yes` / `no`). If they decline, abort without calling delete.

404 at this step means the id is wrong or belongs to a different sub-site — report and abort.

### Step 3: Delete

```bash
python3 skills/korea/scripts/korea_api.py --api-key "{KEY}" \
  --base-url "https://{SUBSITE}/api/v1" \
  banner-delete --id {BANNER_ID}
```

Expected response: `{ "data": { "deleted": true } }`.

### Step 4: Report the result

On success, tell the user: "Banner #{id} ({title}) soft-deleted on {sub-site}. It will no longer render and is hidden from admin listings." Mention that recovery requires a DB update (`UPDATE banner_ads SET deleted_at = NULL WHERE id = ...`).

On 403/404/401, relay the server `message` verbatim.

## Notes

- **Soft-delete only.** The row stays in `banner_ads`; `deleted_at` is stamped. No API path exists to undo.
- **Always confirm with the user** before calling delete. This command has no dry-run mode.
- Deleting a banner **does not** delete its uploaded files — remove them separately via `DELETE /files/{id}` if desired.
