---
name: moodle-hvp-h5p-migration
description: Migrating HVP activities to H5P in Moodle using tool_migratehvp2h5p, including the c=0 vs c=1 permission issue
---

# Moodle HVP → H5P Migration

## Migration tool

Plugin: `admin/tool/migratehvp2h5p`

CLI script:
```bash
php admin/tool/migratehvp2h5p/cli/migrate.php [options]
```

### Key options

| Option | Meaning |
|---|---|
| `-c=0` | **Embed** H5P file directly in the activity (standalone, no Content Bank link) |
| `-c=1` | **Link** to Content Bank — creates file alias; students need `moodle/contentbank:access` |
| `-k=0` | Delete original HVP activities after migration |
| `-k=1` | Keep original HVP activities |
| `--execute` | Actually run (omit for dry run) |

### Recommended command

```bash
php admin/tool/migratehvp2h5p/cli/migrate.php -k=0 -c=0 --execute
```

## ⚠️ c=1 permission bug

**Problem:** When migrated with `-c=1`, H5P activities store the `.h5p` file as an alias pointing to a Content Bank item (`referencefileid IS NOT NULL` in `mdl_files`). Accessing such an activity requires the capability `moodle/contentbank:access`.

The `student` role does **not** have this capability by default → students see the activity but cannot open it.

**DB diagnostic query:**
```sql
SELECT COUNT(DISTINCT ctx.instanceid) AS broken_activities
FROM mdl_files f
JOIN mdl_context ctx ON ctx.id = f.contextid
WHERE ctx.contextlevel = 70   -- module context
  AND f.component = 'mod_h5pactivity'
  AND f.filearea = 'package'
  AND f.filename != '.'
  AND f.referencefileid IS NOT NULL;
```

### Fix options

**Option A — Grant capability (security risk):**
Add `moodle/contentbank:access` to the `student` role. Risk: students can browse draft/unpublished content bank items in their courses.

**Option B — Re-migrate with `-c=0` (recommended, zero risk):**
If original HVP activities still exist (`mdl_hvp` table not empty), re-run migration with `-c=0`. Check first:
```sql
SELECT COUNT(*) FROM mdl_hvp;
```

If HVP records remain, simply re-run:
```bash
php admin/tool/migratehvp2h5p/cli/migrate.php -k=0 -c=0 --execute
```

This replaces the aliased files with embedded copies and students can access without any extra capability.

## Running CLI in Docker

On a Dockerised Moodle (fpm container), run:
```bash
docker compose exec moodle-fpm php admin/tool/migratehvp2h5p/cli/migrate.php -k=0 -c=0 --execute
```

Or via SSH:
```bash
ssh complett-live "docker compose exec moodle-fpm php admin/tool/migratehvp2h5p/cli/migrate.php -k=0 -c=0 --execute"
```

## File reference internals

A c=1 file reference stores a base64-encoded PHP-serialised array in `mdl_files_reference.reference`:
```php
// Decoded structure:
[
  'contextid' => <content bank context id>,
  'component' => 'contentbank',
  'filearea'  => 'public',
  'itemid'    => <content bank item id>,
  'filename'  => 'content.h5p',
]
```

## Useful DB queries

```sql
-- Which users can't access H5P (students missing capability)?
SELECT u.username, r.shortname
FROM mdl_role_assignments ra
JOIN mdl_user u ON u.id = ra.userid
JOIN mdl_role r ON r.id = ra.roleid
WHERE ra.userid = <userid>;

-- Check if student role has contentbank:access
SELECT rc.permission
FROM mdl_role_capabilities rc
JOIN mdl_capabilities cap ON cap.name = rc.capability
JOIN mdl_role r ON r.id = rc.roleid
WHERE cap.name = 'moodle/contentbank:access' AND r.shortname = 'student';
-- Empty result = capability not set = default (not allowed)
```
