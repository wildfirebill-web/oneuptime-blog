# How to Use redis-check-rdb for RDB File Validation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RDB, Backup

Description: Learn how to use redis-check-rdb to validate and inspect Redis RDB snapshot files before restoring them to avoid data corruption issues.

---

`redis-check-rdb` is a utility shipped with Redis that validates RDB (Redis Database) snapshot files. Before restoring a backup, running this tool can save you from loading a corrupted file into production.

## What is an RDB File?

Redis periodically saves the dataset to disk in a binary format called RDB (Redis Database). These files are created by:

- The `SAVE` or `BGSAVE` command
- Automatic background saving based on `save` directives in `redis.conf`
- Replication sync snapshots

```bash
# Trigger an RDB save manually
redis-cli BGSAVE
redis-cli LASTSAVE  # timestamp of last successful save
ls -lh /var/lib/redis/dump.rdb
```

## Running redis-check-rdb

Basic validation of an RDB file:

```bash
redis-check-rdb /var/lib/redis/dump.rdb
```

Successful output:

```text
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '7.0.5'
[offset 40] AUX FIELD redis-bits = '64'
[offset 52] AUX FIELD ctime = '1706745600'
[offset 61] AUX FIELD used-mem = '2097152'
[offset 78] AUX FIELD aof-base = '0'
[offset 80] Selecting DB ID 0
[offset 13421] Checksum OK
```

If the file is corrupted:

```text
[offset 0] Checking RDB file dump.rdb
[offset 26] AUX FIELD redis-ver = '7.0.5'
[offset 12045] FATAL: RDB checksum error
```

## Validating Before Restore

Always validate before restoring:

```bash
#!/bin/bash
RDB_FILE="$1"
if redis-check-rdb "$RDB_FILE" 2>&1 | grep -q "Checksum OK"; then
  echo "RDB file is valid. Proceeding with restore."
  cp "$RDB_FILE" /var/lib/redis/dump.rdb
  chown redis:redis /var/lib/redis/dump.rdb
else
  echo "ERROR: RDB file is corrupted. Restore aborted."
  exit 1
fi
```

## Inspecting RDB File Metadata

The tool prints metadata embedded in the RDB file:

```bash
redis-check-rdb /backups/redis-2026-03-31.rdb 2>&1 | grep "AUX FIELD"
```

Output shows:

```text
[offset 26] AUX FIELD redis-ver = '7.0.5'
[offset 40] AUX FIELD redis-bits = '64'
[offset 52] AUX FIELD ctime = '1711843200'
```

Use the `ctime` value to confirm the snapshot timestamp:

```bash
date -d @1711843200
```

## Testing RDB Compatibility

When migrating between Redis versions, validate the RDB format compatibility:

```bash
# Check RDB version (first 9 bytes)
xxd /var/lib/redis/dump.rdb | head -1
# REDIS0011 means RDB version 11 (Redis 7.x)
```

```bash
# Full validation with output capture
redis-check-rdb /var/lib/redis/dump.rdb > /tmp/rdb-check.log 2>&1
cat /tmp/rdb-check.log
```

## Automating RDB Validation in Backup Pipelines

```bash
#!/bin/bash
BACKUP_DIR="/backups/redis"
FAILED=0

for rdb in "$BACKUP_DIR"/*.rdb; do
  if redis-check-rdb "$rdb" 2>&1 | grep -q "Checksum OK"; then
    echo "OK: $rdb"
  else
    echo "FAIL: $rdb"
    FAILED=$((FAILED + 1))
  fi
done

exit $FAILED
```

## Summary

`redis-check-rdb` is an essential utility for validating Redis RDB snapshot files before restoring them. It checks the file checksum, reports metadata, and clearly indicates corruption. Integrating this check into your backup and restore pipelines prevents data loss from bad snapshots.
