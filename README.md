# wp-backup: WordPress Backup & Restore for DDEV

A comprehensive backup and restore system for WordPress sites running in DDEV. Create scoped backups of your database and files, then restore them with a single command.

## Requirements

**Required:**
- `jq` - JSON processor for managing backup metadata
  ```bash
  sudo apt install jq
  ```

**Optional:**
- `pv` - Shows transfer rate and throughput during backup/restore operations
  ```bash
  sudo apt install pv
  ```
- `pigz` - Parallel gzip for faster compression (uses multiple CPU cores)
  ```bash
  sudo apt install pigz
  ```

## Quick Start

```bash
# Create a backup of your entire site
ddev wp-backup

# Create a named backup before updating plugins
ddev wp-backup plugins -n before_update

# List all backups
ddev wp-backup list

# Restore the latest backup
ddev wp-backup restore

# Restore a specific named backup
ddev wp-backup restore -n before_update
```

## Understanding Scopes

Scopes determine what gets backed up. All scopes include the database plus specified files:

| Scope | Database | Files Included |
|-------|----------|----------------|
| `db` | ✓ | None (database only) |
| `plugins` | ✓ | wp-content/plugins |
| `themes` | ✓ | wp-content/themes |
| `extensions` | ✓ | plugins, themes, mu-plugins & drop-ins |
| `full` | ✓ | Entire site **[DEFAULT]** |

**When to use `db` scope?** Perfect for quick database snapshots when you only need to preserve data, not files. Much faster than other scopes.

**When to use `extensions` scope?** Great for backing up all WordPress extensions (plugins, themes, mu-plugins, and drop-in files like `object-cache.php`) without including uploads or other site files.

## Creating Backups

### Basic Backup (Default Scope)
```bash
ddev wp-backup
```
Creates a backup of your database and entire site.

### Scope-Specific Backups
```bash
# Quick database snapshot
ddev wp-backup db

# Before updating plugins
ddev wp-backup plugins

# Before changing themes
ddev wp-backup themes

# All extensions (plugins, themes, mu-plugins, drop-ins)
ddev wp-backup extensions
```

### Named Backups
Names help identify backups. They're automatically sanitized (lowercase, underscores for spaces):

```bash
ddev wp-backup plugins -n before_update
ddev wp-backup plugins --name "Before Update"  # Same as above
```

**Multiple backups can share the same name** - the timestamp makes each unique. When restoring by name, you'll get the most recent one.

## Listing Backups

### List All Backups
```bash
ddev wp-backup list
```

### Filter by Scope
```bash
ddev wp-backup list plugins
ddev wp-backup list full
```

Example output:
```
Available Backups:

SCOPE         NAME                            SIZE      CREATED
────────────  ──────────────────────────────  ────────  ────────────────────
plugins       before_update                   45MB      2025-12-07 09:18:55
full          -                               120MB     2025-12-07 08:30:12
extensions    pre_migration                   85MB      2025-12-06 15:22:03
```

**Note:** The `-` in the NAME column indicates an unnamed backup.

## Restoring Backups

### Restore Priority

When you restore, the command searches for backups in this order:

1. **By ID** (exact match)
2. **By scope + name** (latest matching both)
3. **By name only** (latest with that name, any scope)
4. **By scope only** (latest for that scope)
5. **Latest backup** (most recent, any scope)

### Restore Latest Backup
```bash
# Restore the most recent backup (any scope)
ddev wp-backup restore
```

### Restore by Scope
```bash
# Restore the latest plugins backup
ddev wp-backup restore plugins

# Restore the latest full backup
ddev wp-backup restore full
```

### Restore by Name
```bash
# Restore latest backup named 'before_update' (any scope)
ddev wp-backup restore -n before_update

# Restore latest plugins backup named 'before_update'
ddev wp-backup restore plugins -n before_update
```

### Restore by ID
When multiple backups share a name, use the specific ID:

```bash
# Get the ID from 'ddev wp-backup list'
ddev wp-backup restore -i plugins-20251207-091855-before_update
```

### What Gets Deleted During Restore?

The restore process **deletes files** based on scope before extracting the backup:

| Scope | What Gets Deleted |
|-------|------------------|
| `db` | Nothing (database only) |
| `plugins` | wp-content/plugins |
| `themes` | wp-content/themes |
| `extensions` | plugins, themes, mu-plugins & drop-in PHP files |
| `full` | Everything except .ddev |

**You will be prompted to confirm** before any files are deleted.

## Cleanup

### Remove All Backups
```bash
ddev wp-backup cleanup
```

### Remove Backups by Scope
```bash
# Remove all plugins backups
ddev wp-backup cleanup plugins

# Remove all full backups
ddev wp-backup cleanup full
```

### Remove Backups by Name
```bash
# Remove all backups named 'before_update' (any scope)
ddev wp-backup cleanup before_update
```

### Keep Only Recent Backups

The `--keep N` flag lets you retain the N newest backups and delete the rest:

```bash
# Keep the newest 3 backups of each scope, delete the rest
ddev wp-backup cleanup --keep 3

# Keep the newest 2 plugin backups, delete older plugin backups
ddev wp-backup cleanup plugins --keep 2

# Keep only the most recent backup of each scope
ddev wp-backup cleanup --keep 1
```

When using `--keep`, you'll see a detailed confirmation showing which backups will be kept and which will be deleted:

```
⚠ WARNING: Will delete 3 of 5 'plugins' backups (keeping newest 2)

Keeping:
  - plugins-20251207-091855-before_update
  - plugins-20251206-100000-recent

Deleting:
  - plugins-20251205-100000-old1
  - plugins-20251204-100000-old2
  - plugins-20251203-100000-old3

Proceed? [y/N]
```

**Note:** The `--keep` flag only works with scopes, not backup names. Cleanup deletes both backup files and removes entries from the index. You'll be prompted to confirm.

## Advanced Features

### Rebuild Index

If your backup index becomes corrupted or out of sync:

```bash
ddev wp-backup rebuild
```

This scans all backup files in `.ddev/backups/` and reconstructs the index. Useful for:
- Recovering from a corrupted index.json
- Importing backups created on another machine
- Fixing mismatched metadata

### Backup Verification

All backups are automatically verified after creation:
- Gzip integrity test
- Tar archive integrity test
- File size validation (ensures non-empty files)

If verification fails, the backup files are deleted and you'll see an error.

## Common Workflows

### Quick Database Snapshot
```bash
# Fast database backup before making data changes
ddev wp-backup db -n before_content_import

# Import content or make database changes...
# To roll back:

ddev wp-backup restore db -n before_content_import
```

### Before Plugin Update
```bash
# Create a backup
ddev wp-backup plugins -n before_acf_update

# Update the plugin...
# If something breaks:

ddev wp-backup restore plugins -n before_acf_update
```

### Before Major Changes
```bash
# Full backup before big changes
ddev wp-backup full -n before_redesign

# Make changes...
# To roll back:

ddev wp-backup restore full -n before_redesign
```

### Regular Development Snapshots
```bash
# Full site backup (default)
ddev wp-backup -n working_state

# Later, if you need to reset:
ddev wp-backup restore -n working_state
```

### Testing Theme Changes
```bash
# Backup current theme
ddev wp-backup themes -n before_customization

# Test changes...
# Not happy? Restore:

ddev wp-backup restore themes -n before_customization
```

### Managing Disk Space
```bash
# Keep only the 3 most recent backups of each scope
ddev wp-backup cleanup --keep 3

# Or keep different amounts per scope
ddev wp-backup cleanup plugins --keep 5   # Keep 5 plugin backups
ddev wp-backup cleanup full --keep 1      # Keep only 1 full backup (they're large!)
ddev wp-backup cleanup db --keep 10       # Keep 10 db backups (they're small)
```

## File Locations

All backups are stored in your project's `.ddev/backups/` directory:

```
.ddev/backups/
├── index.json                                    # Metadata tracking
├── plugins-20251207-091855-before_update-files.tar.gz
├── plugins-20251207-091855-before_update-db.sql.gz
├── full-20251207-083012--files.tar.gz           # Unnamed backup
└── full-20251207-083012--db.sql.gz
```

Each backup consists of:
- **{id}-files.tar.gz** - Compressed filesystem backup (not created for `db` scope)
- **{id}-db.sql.gz** - Compressed database dump
- **index.json** entry with metadata

## Troubleshooting

### "Error: jq is required but not installed"
Install jq:
```bash
sudo apt install jq
```

### "No matching backup found"
- Run `ddev wp-backup list` to see available backups
- Check that you're using the correct scope or name
- Verify backup files exist in `.ddev/backups/`

### Index Out of Sync
If backups exist but don't appear in listings:
```bash
ddev wp-backup rebuild
```

### Backup Verification Failed
This usually indicates:
- Insufficient disk space
- Permission issues
- Interrupted backup process

Check disk space with `df -h` and try again.

## Help

View built-in help:
```bash
ddev wp-backup --help
```

## Best Practices

1. **Name your backups** - Future you will thank present you
2. **Use specific scopes** - Faster backups and restores
3. **Regular cleanups** - Use `--keep N` to automatically retain recent backups while removing old ones
4. **Test restores** - Verify backups work before you need them
5. **Before major changes** - Always create a backup first
