# How to Recover from Redis Data Loss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Disaster Recovery, Data Recovery, Persistence, DevOps

Description: Learn step-by-step how to recover from Redis data loss using RDB snapshots, AOF files, and backup restoration procedures to minimize data loss impact.

---

Redis data loss can occur due to hardware failure, accidental FLUSHALL, corrupted persistence files, or a misconfigured eviction policy. Recovery options depend on what persistence mechanisms were enabled before the incident.

## Assess the Situation

Before attempting recovery, gather information:

```bash
# Check if Redis is still running
redis-cli PING

# Check current key count
redis-cli DBSIZE

# Check persistence configuration
redis-cli CONFIG GET save
redis-cli CONFIG GET appendonly

# Check last RDB save time
redis-cli INFO persistence | grep -E "rdb_last_save_time|aof_enabled"
```

## Recovery Option 1: Restore from RDB Snapshot

RDB is the fastest recovery path. The RDB file is a point-in-time snapshot:

```bash
# Locate existing RDB file
ls -lh /var/lib/redis/dump.rdb

# Check its modification time (this is your recovery point)
date -r /var/lib/redis/dump.rdb

# Stop Redis before restoring
systemctl stop redis

# If the current dump.rdb is corrupted, check its integrity
redis-check-rdb /var/lib/redis/dump.rdb

# Restore from a backup copy
cp /backups/redis/dump-2026-03-31.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# Start Redis
systemctl start redis

# Verify key count
redis-cli DBSIZE
```

## Recovery Option 2: Recover from AOF File

AOF (Append Only File) provides finer-grained recovery since it logs every write operation:

```bash
# Check AOF file location
redis-cli CONFIG GET dir
redis-cli CONFIG GET appendfilename

# Verify AOF integrity
redis-check-aof /var/lib/redis/appendonly.aof

# Repair a corrupted AOF (truncates at the point of corruption)
redis-check-aof --fix /var/lib/redis/appendonly.aof
```

To replay the AOF up to a specific point, you can truncate it before the problematic operation:

```bash
# Find the FLUSHALL command in the AOF (if accidental flush occurred)
grep -n "FLUSHALL" /var/lib/redis/appendonly.aof

# Truncate the AOF file before the FLUSHALL line
head -n 1420 /var/lib/redis/appendonly.aof > /tmp/appendonly-recovered.aof
cp /tmp/appendonly-recovered.aof /var/lib/redis/appendonly.aof
```

## Recovery Option 3: Restore from Remote Backup

If local files are lost, restore from your backup storage:

```bash
#!/bin/bash
BACKUP_DATE="${1:-$(date +%Y-%m-%d)}"
BUCKET="s3://my-redis-backups"

# List available backups
aws s3 ls "$BUCKET/daily/" | grep "$BACKUP_DATE"

# Download the desired backup
aws s3 cp "$BUCKET/daily/redis-backup-${BACKUP_DATE}.rdb" /tmp/restore.rdb

# Stop Redis, restore, and start
systemctl stop redis
cp /tmp/restore.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb
systemctl start redis

echo "Restored. Key count: $(redis-cli DBSIZE)"
```

## Recovery Option 4: For Caches - Warm From Database

If Redis is used purely as a cache with no critical data, the fastest recovery is warming the cache from your source database rather than restoring backups:

```python
import redis
import psycopg2

r = redis.Redis()
db = psycopg2.connect("dbname=myapp")

# Warm session cache from database
cursor = db.cursor()
cursor.execute("SELECT session_id, data FROM sessions WHERE expires_at > NOW()")
pipe = r.pipeline()
for session_id, data in cursor.fetchall():
    pipe.set(f"session:{session_id}", data, ex=3600)
    if len(pipe.command_stack) >= 500:
        pipe.execute()
pipe.execute()
print(f"Warmed {cursor.rowcount} sessions")
```

## Preventing Future Data Loss

After recovery, harden your persistence configuration:

```bash
# Enable both RDB and AOF
redis-cli CONFIG SET save "900 1 300 10 60 10000"
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
redis-cli CONFIG SET no-appendfsync-on-rewrite no

# Save config permanently
redis-cli CONFIG REWRITE
```

## Summary

Redis data loss recovery depends on what persistence was enabled. RDB snapshots provide fast point-in-time recovery, while AOF files allow more granular recovery, including truncating before a destructive operation like FLUSHALL. Always test your recovery procedures before an incident and configure both RDB and AOF for production workloads requiring durability.
