# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a DDEV custom commands repository. The primary command is `wp-backup`, a comprehensive WordPress backup/restore system designed for DDEV environments.

## wp-backup Architecture

### Core Concepts

**Scope System**: Seven backup scopes that determine what gets backed up:
- `db`: Database only (no files)
- `plugins`: DB + wp-content/plugins
- `themes`: DB + wp-content/themes
- `wp-content`: DB + wp-content (excluding uploads)
- `uploads`: DB + wp-content/uploads
- `site`: DB + entire site (excluding uploads) - **default scope**
- `full`: DB + entire site (including uploads)

**Backup Components**: Each backup typically consists of two files:
- `{id}-files.tar.gz`: Filesystem backup (not created for `db` scope)
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
3. Creates filesystem backup (tar.gz) - skipped for `db` scope
4. Exports database (via `ddev export-db`)
5. Verifies integrity (`verify_backup_files`) - scope-aware
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
- Optional `--keep N` flag to retain the N newest backups
- When `--keep` is used with a scope: keeps N newest of that scope
- When `--keep` is used without a scope: keeps N newest of each scope
- Deletes files and removes from index
- Shows detailed confirmation with "Keeping" and "Deleting" lists when using `--keep`

**Rebuild Index** (`rebuild_index_from_files`):
- Reconstructs index.json from existing backup files
- Useful for recovery if index becomes corrupted

### Important Functions

**`build_tar_command(scope)`**: Sets `INCLUDE` and `EXCLUDE` arrays based on scope. Critical for ensuring correct files are backed up/restored.

**`verify_backup_files(files_tar, db_dump, scope)`**: Tests gzip and tar integrity, checks for empty files. Prevents corrupted backups from being saved. Skips file verification for `db` scope.

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

Scope-aware deletion during restore (lines 580-656):
- `db`: No file deletion (database only)
- `plugins`: Deletes wp-content/plugins
- `themes`: Deletes wp-content/themes
- `wp-content`: Deletes wp-content but preserves uploads (moved temporarily)
- `uploads`: Deletes wp-content/uploads
- `site`: Deletes all files except .ddev and preserves wp-content/uploads
- `full`: Deletes all files except .ddev

### Dependencies

**Required**:
- `jq`: JSON parsing for index management. Script exits if not found.

**Optional**:
- `pv`: Shows transfer rate and throughput during backup/restore operations. Displays accurate progress bars for restore (reading compressed files), but shows throughput metrics only for backup creation (due to tar streaming and compression). Script detects and uses if available (`HAS_PV` variable).
- `pigz`: Parallel gzip implementation for faster compression using multiple CPU cores. Script automatically detects and uses if available (`GZIP_CMD` variable). Falls back to standard `gzip` if not installed.

### File Paths

```bash
PROJECT_ROOT="$(pwd)"                      # Current directory
BACKUP_DIR="${PROJECT_ROOT}/.ddev/backups" # Backup storage
INDEX_FILE="${BACKUP_DIR}/index.json"      # Metadata index
```

### Common Development Patterns

**When modifying backup scopes**:
1. Update `valid_scopes` array (line 106)
2. Add case to `build_tar_command` (lines 257-294)
3. Add case to scope-aware deletion in `restore_backup` (lines 580-656)
4. Update help text in `show_help` (lines 137-144)
5. Update argument parser to include new scope (line 1026)

**When adding new operations**:
- Add case to main argument parser (lines 936-1051)
- Follow pattern of existing operations for consistency
- Always validate scope with `is_valid_scope` before use

### Edge Cases & Gotchas

1. **Empty Names**: Uses `-` as placeholder to avoid parsing issues in tab-delimited output
2. **Duplicate Names**: Multiple backups can share the same name; ID uniquely identifies each
3. **Stale Index**: `list_backups` auto-cleans entries where files are missing
4. **wp-content Restore**: Special logic to preserve uploads when restoring wp-content scope
5. **User Confirmation**: All destructive operations (restore, cleanup) require explicit confirmation
6. **DB-Only Backups**: The `db` scope creates only a database dump (no files tar), useful for quick database snapshots

### Testing Changes

When modifying wp-backup:

```bash
# Test backup creation
ddev wp-backup plugins -n test_backup
ddev wp-backup db -n test_db

# Verify index
cat .ddev/backups/index.json | jq

# Test listing
ddev wp-backup list
ddev wp-backup list plugins
ddev wp-backup list db

# Test restore (use test environment!)
ddev wp-backup restore plugins -n test_backup
ddev wp-backup restore db -n test_db

# Test cleanup
ddev wp-backup cleanup test_backup
ddev wp-backup cleanup --keep 2
ddev wp-backup cleanup plugins --keep 1
```

### Shell Best Practices Used

- `set -euo pipefail`: Strict error handling
- Proper quoting of variables to handle spaces
- Array usage for building tar arguments
- `shopt -s dotglob nullglob` for safe file iteration
- Temporary file pattern for atomic index updates (`.tmp` files)
