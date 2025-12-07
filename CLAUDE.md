# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a DDEV custom commands repository. The primary command is `wp-backup`, a comprehensive WordPress backup/restore system designed for DDEV environments.

## wp-backup Architecture

### Core Concepts

**Scope System**: Six backup scopes that determine what gets backed up:
- `plugins`: DB + wp-content/plugins
- `themes`: DB + wp-content/themes
- `wp-content`: DB + wp-content (excluding uploads)
- `uploads`: DB + wp-content/uploads
- `site`: DB + entire site (excluding uploads) - **default scope**
- `full`: DB + entire site (including uploads)

**Backup Components**: Each backup consists of two files:
- `{id}-files.tar.gz`: Filesystem backup
- `{id}-db.sql.gz`: Database dump

**ID Format**: `{scope}-{YYYYMMDD}-{HHMMSS}-{name}`
- Example: `plugins-20251207-091855-before_update`
- Name is optional; uses `-` as placeholder when empty

**Metadata Index**: `.ddev/backups/index.json` tracks all backups with:
```json
{
  "id": "plugins-20251207-091855-before_update",
  "scope": "plugins",
  "name": "before_update",
  "timestamp": "2025-12-07T09:18:55+00:00",
  "files": "plugins-20251207-091855-before_update-files.tar.gz",
  "db": "plugins-20251207-091855-before_update-db.sql.gz"
}
```

### Key Operations

**Create Backup** (`create_backup`):
1. Validates scope
2. Builds tar command with scope-specific includes/excludes
3. Creates filesystem backup (tar.gz)
4. Exports database (via `ddev export-db`)
5. Verifies integrity (`verify_backup_files`)
6. Adds metadata to index

**Restore Backup** (`restore_backup`):
- **Priority Order**: ID > (scope + name) > name > scope > latest
- Multiple backups can share the same name; restore uses most recent
- Scope-aware deletion with user confirmation
- Special handling for wp-content scope (preserves uploads during restore)

**List Backups** (`list_backups`):
- Auto-cleans stale index entries (missing files)
- Calculates and displays total size
- Can filter by scope

**Cleanup** (`cleanup_backups`):
- Can target all backups, specific scope, or specific name
- Deletes files and removes from index

**Rebuild Index** (`rebuild_index_from_files`):
- Reconstructs index.json from existing backup files
- Useful for recovery if index becomes corrupted

### Important Functions

**`build_tar_command(scope)`**: Sets `INCLUDE` and `EXCLUDE` arrays based on scope. Critical for ensuring correct files are backed up/restored.

**`verify_backup_files(files_tar, db_dump)`**: Tests gzip and tar integrity, checks for empty files. Prevents corrupted backups from being saved.

**`sanitize_name(name)`**: Converts backup names to lowercase, removes special chars, replaces spaces with underscores. Ensures consistent naming.

**`find_latest_backup(scope)`**: Queries index for most recent backup, optionally filtered by scope. Uses timestamp sorting.

### Restoration Logic

The restore priority system is key to understanding behavior:

```bash
# Priority: ID > (scope + name) > name > scope > latest
if [ -n "$id" ]; then
    # Exact ID match
elif [ -n "$name" ] && [ -n "$scope" ]; then
    # Latest backup matching both scope AND name
elif [ -n "$name" ]; then
    # Latest backup matching name (any scope)
elif [ -n "$scope" ]; then
    # Latest backup for scope
else
    # Latest backup overall
fi
```

Scope-aware deletion during restore (lines 466-515):
- `plugins`: Deletes wp-content/plugins
- `themes`: Deletes wp-content/themes
- `wp-content`: Deletes wp-content but preserves uploads (moved temporarily)
- `uploads`: Deletes wp-content/uploads
- `site`/`full`: Deletes all files except .ddev

### Dependencies

**Required**:
- `jq`: JSON parsing for index management. Script exits if not found.

**Optional**:
- `pv`: Progress bars for tar operations. Script detects and uses if available (`HAS_PV` variable).

### File Paths

```bash
PROJECT_ROOT="$(pwd)"                      # Current directory
BACKUP_DIR="${PROJECT_ROOT}/.ddev/backups" # Backup storage
INDEX_FILE="${BACKUP_DIR}/index.json"      # Metadata index
```

### Common Development Patterns

**When modifying backup scopes**:
1. Update `valid_scopes` array (line 100)
2. Add case to `build_tar_command` (lines 177-211)
3. Add case to scope-aware deletion in `restore_backup` (lines 466-515)
4. Update help text in `show_help` (lines 130-137)

**When adding new operations**:
- Add case to main argument parser (lines 670-740)
- Follow pattern of existing operations for consistency
- Always validate scope with `is_valid_scope` before use

### Edge Cases & Gotchas

1. **Empty Names**: Uses `-` as placeholder to avoid parsing issues in tab-delimited output
2. **Duplicate Names**: Multiple backups can share the same name; ID uniquely identifies each
3. **Stale Index**: `list_backups` auto-cleans entries where files are missing
4. **wp-content Restore**: Special logic to preserve uploads when restoring wp-content scope
5. **User Confirmation**: All destructive operations (restore, cleanup) require explicit confirmation

### Testing Changes

When modifying wp-backup:

```bash
# Test backup creation
ddev wp-backup plugins -n test_backup

# Verify index
cat .ddev/backups/index.json | jq

# Test listing
ddev wp-backup list
ddev wp-backup list plugins

# Test restore (use test environment!)
ddev wp-backup restore plugins -n test_backup

# Test cleanup
ddev wp-backup cleanup test_backup
```

### Shell Best Practices Used

- `set -euo pipefail`: Strict error handling
- Proper quoting of variables to handle spaces
- Array usage for building tar arguments
- `shopt -s dotglob nullglob` for safe file iteration
- Temporary file pattern for atomic index updates (`.tmp` files)
