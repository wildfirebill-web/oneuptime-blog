# How to Write a Redis Key Migration Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Migration, Script, Keys, Data Transfer

Description: Migrate Redis keys between instances using DUMP/RESTORE, MIGRATE, and pattern-based SCAN without downtime or data loss.

---

Redis key migrations are needed when moving to a new server, splitting a large database, or changing a key naming convention. Redis provides built-in commands for atomic key transfer, and a migration script orchestrates them safely with progress tracking and error handling.

## MIGRATE Command - Simplest Approach

`MIGRATE` atomically moves a key from one Redis instance to another:

```bash
# Migrate a single key
redis-cli -h source-host MIGRATE dest-host 6379 mykey 0 5000

# Migrate with authentication on destination
redis-cli -h source-host MIGRATE dest-host 6379 mykey 0 5000 AUTH dest-password

# Migrate without deleting from source (COPY flag)
redis-cli -h source-host MIGRATE dest-host 6379 mykey 0 5000 COPY
```

## Bash Migration Script with SCAN

Migrate all keys matching a pattern from source to destination:

```bash
#!/bin/bash
# redis-migrate.sh

SRC_HOST="${SRC_HOST:-localhost}"
SRC_PORT="${SRC_PORT:-6379}"
SRC_PASS="${SRC_PASS:-}"
DST_HOST="${DST_HOST:-localhost}"
DST_PORT="${DST_PORT:-6380}"
DST_PASS="${DST_PASS:-}"
PATTERN="${PATTERN:-*}"
TIMEOUT=5000
DRY_RUN="${DRY_RUN:-true}"
BATCH_SIZE=100

src_cli() {
    local args=("-h" "$SRC_HOST" "-p" "$SRC_PORT" "--no-auth-warning")
    [ -n "$SRC_PASS" ] && args+=("-a" "$SRC_PASS")
    redis-cli "${args[@]}" "$@"
}

log() { echo "[$(date '+%H:%M:%S')] $*"; }

migrated=0
failed=0
cursor=0

log "Starting migration: $SRC_HOST:$SRC_PORT -> $DST_HOST:$DST_PORT (pattern=$PATTERN)"

while true; do
    result=$(src_cli SCAN "$cursor" MATCH "$PATTERN" COUNT "$BATCH_SIZE")
    cursor=$(echo "$result" | head -1)
    keys=$(echo "$result" | tail -n +2)

    for key in $keys; do
        if [ "$DRY_RUN" = "true" ]; then
            log "[DRY] Would migrate: $key"
            migrated=$((migrated + 1))
        else
            MIGRATE_ARGS=("$DST_HOST" "$DST_PORT" "$key" "0" "$TIMEOUT")
            [ -n "$DST_PASS" ] && MIGRATE_ARGS+=("AUTH" "$DST_PASS")

            result=$(src_cli MIGRATE "${MIGRATE_ARGS[@]}" 2>&1)
            if [ "$result" = "OK" ] || [ "$result" = "NOKEY" ]; then
                migrated=$((migrated + 1))
            else
                log "FAILED to migrate $key: $result"
                failed=$((failed + 1))
            fi
        fi
    done

    [ "$cursor" = "0" ] && break
    sleep 0.005  # rate limit
done

log "Migration complete: migrated=$migrated failed=$failed"
```

## Python Migration with DUMP/RESTORE

For cross-version migrations where MIGRATE may not work, use DUMP/RESTORE:

```python
#!/usr/bin/env python3
import redis
import os

src = redis.Redis(host=os.getenv("SRC_HOST", "localhost"),
                  port=int(os.getenv("SRC_PORT", 6379)),
                  password=os.getenv("SRC_PASS"), decode_responses=False)

dst = redis.Redis(host=os.getenv("DST_HOST", "localhost"),
                  port=int(os.getenv("DST_PORT", 6380)),
                  password=os.getenv("DST_PASS"), decode_responses=False)

def migrate_key(key: bytes) -> bool:
    try:
        data = src.dump(key)
        if data is None:
            return False
        ttl = src.pttl(key)
        ttl = ttl if ttl > 0 else 0
        dst.restore(key, ttl, data, replace=True)
        return True
    except Exception as e:
        print(f"Error migrating {key}: {e}")
        return False

def migrate_pattern(pattern: str = "*"):
    migrated = 0
    cursor = 0
    while True:
        cursor, keys = src.scan(cursor, match=pattern.encode(), count=200)
        for key in keys:
            if migrate_key(key):
                migrated += 1
        if cursor == 0:
            break
    return migrated

if __name__ == "__main__":
    pattern = os.getenv("PATTERN", "*")
    count = migrate_pattern(pattern)
    print(f"Migrated {count} keys")
```

## Summary

Redis key migrations use MIGRATE for live atomic transfers between instances, or DUMP/RESTORE for cross-version compatibility. A SCAN-based migration script processes keys in batches with rate limiting to avoid impacting production. Dry-run mode previews which keys will be moved, and failed key tracking ensures you can retry or investigate errors after the run.
