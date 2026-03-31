# Redis Runbook: Handling Persistence Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Persistence, Runbook, Operations, RDB, AOF, Disaster Recovery

Description: A step-by-step operations runbook for diagnosing and recovering from Redis RDB and AOF persistence failures to prevent data loss in production.

---

## Overview

Redis persistence failures can manifest as failed background saves, corrupted RDB files, or AOF rewrite errors. Left unaddressed, they can lead to data loss on restart. This runbook provides actionable steps to diagnose and recover from each category of persistence failure.

## Detecting Persistence Failures

### Check Persistence Status via INFO

```bash
redis-cli INFO persistence
```

```text
rdb_changes_since_last_save:15234
rdb_bgsave_in_progress:0
rdb_last_save_time:1711900000
rdb_last_bgsave_status:err
rdb_last_bgsave_time_sec:-1
rdb_last_cow_size:0
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_size:104857600
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
```

Key fields to monitor:
- `rdb_last_bgsave_status`: `ok` or `err`
- `aof_last_bgrewrite_status`: `ok` or `err`
- `aof_last_write_status`: `ok` or `err`
- `rdb_changes_since_last_save`: high numbers indicate a save has been failing

### Check Logs for Errors

```bash
journalctl -u redis -n 200 | grep -iE "error|fail|warn|persistence|rdb|aof"
```

Common error messages:
- `Can't save in background: fork: Cannot allocate memory`
- `Error moving temp DB file on the final place`
- `AOF rewrite: error writing or reading the append only file`
- `MISCONF Redis is configured to save RDB snapshots`

## RDB Persistence Failures

### Scenario 1: BGSAVE Fails with Fork Error

**Symptom:** `rdb_last_bgsave_status:err`, log shows `fork: Cannot allocate memory`

**Cause:** Redis uses copy-on-write during BGSAVE. If `overcommit_memory` is not enabled and available memory is low, fork fails.

**Resolution:**

```bash
# Check current overcommit setting
cat /proc/sys/vm/overcommit_memory

# Enable overcommit (0=heuristic, 1=always allow, 2=deny if exceeds)
echo 1 > /proc/sys/vm/overcommit_memory

# Make it permanent
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
sysctl -p

# Retry the save manually
redis-cli BGSAVE
redis-cli LASTSAVE  # timestamp of last successful save
```

### Scenario 2: Disk Full - RDB Write Fails

**Symptom:** `rdb_last_bgsave_status:err`, disk at 100%

```bash
# Check disk usage
df -h /var/lib/redis

# Find large files
du -sh /var/lib/redis/* | sort -rh | head -10

# Free space by removing old AOF segments or temp files
ls -la /var/lib/redis/*.tmp 2>/dev/null && rm /var/lib/redis/*.tmp

# Verify Redis can write
redis-cli CONFIG GET dir
redis-cli CONFIG GET dbfilename
touch /var/lib/redis/test-write && rm /var/lib/redis/test-write
```

After freeing space, trigger a new save:

```bash
redis-cli BGSAVE
```

### Scenario 3: stop-writes-on-bgsave-error Blocking Writes

**Symptom:** All write commands return: `MISCONF Redis is configured to save RDB snapshots...`

By default, `stop-writes-on-bgsave-error yes` causes Redis to reject writes after a failed BGSAVE. This is a safety mechanism.

```bash
# Temporarily allow writes while you fix the underlying issue
redis-cli CONFIG SET stop-writes-on-bgsave-error no

# Fix the root cause (disk space, memory, permissions)
# Then re-enable for safety
redis-cli CONFIG SET stop-writes-on-bgsave-error yes

# Trigger a successful save to clear the error state
redis-cli BGSAVE
```

### Scenario 4: Corrupted RDB File

If Redis fails to start because the RDB file is corrupted:

```bash
# Check file integrity
redis-check-rdb /var/lib/redis/dump.rdb

# If corrupted, try to recover by removing it (data loss on restart)
mv /var/lib/redis/dump.rdb /var/lib/redis/dump.rdb.corrupted

# Redis will start with an empty dataset
systemctl start redis
```

If a backup exists, restore it:

```bash
systemctl stop redis
cp /backup/redis/dump.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb
systemctl start redis
```

## AOF Persistence Failures

### Scenario 5: AOF Rewrite Fails

```bash
redis-cli INFO persistence | grep aof_last_bgrewrite_status
# aof_last_bgrewrite_status:err

# Check available disk space
df -h $(redis-cli CONFIG GET dir | tail -1)

# Manually trigger AOF rewrite after fixing disk space
redis-cli BGREWRITEAOF
```

### Scenario 6: Corrupted AOF File

Redis will refuse to start if the AOF file has a truncated or corrupted entry.

```bash
# Check the AOF for errors
redis-check-aof /var/lib/redis/appendonly.aof
```

```text
AOF analyzed: size=104857600, ok_up_to=104857400, diff=200
This will truncate the AOF from offset 104857400 to remove invalid data.
```

Fix it:

```bash
# Fix the AOF by truncating at the last valid entry
redis-check-aof --fix /var/lib/redis/appendonly.aof

# Restart Redis
systemctl start redis
```

### Scenario 7: AOF Write Errors Under High Load

**Symptom:** `aof_last_write_status:err` during high write traffic with `appendfsync always`

```bash
# Switch to less aggressive fsync policy temporarily
redis-cli CONFIG SET appendfsync everysec

# Monitor for write errors
redis-cli INFO persistence | grep aof_last_write_status

# Check for I/O saturation
iostat -x 1 5
```

## Monitoring and Alerting

Add these checks to your monitoring system:

```python
import redis

def check_persistence_health(host='localhost', port=6379):
    r = redis.Redis(host=host, port=port, decode_responses=True)
    info = r.info('persistence')

    alerts = []

    if info['rdb_last_bgsave_status'] != 'ok':
        alerts.append(f"RDB BGSAVE failed: {info['rdb_last_bgsave_status']}")

    if info.get('aof_enabled') and info['aof_last_bgrewrite_status'] != 'ok':
        alerts.append(f"AOF rewrite failed: {info['aof_last_bgrewrite_status']}")

    if info.get('aof_enabled') and info['aof_last_write_status'] != 'ok':
        alerts.append(f"AOF write failed: {info['aof_last_write_status']}")

    last_save = info['rdb_last_save_time']
    import time
    age_hours = (time.time() - last_save) / 3600
    if age_hours > 2:
        alerts.append(f"Last RDB save was {age_hours:.1f} hours ago")

    if alerts:
        for alert in alerts:
            print(f"ALERT: {alert}")
        return False

    print("Persistence health: OK")
    return True
```

## Preventing Future Failures

```bash
# Recommended persistence configuration
redis-cli CONFIG SET save "3600 1 300 100 60 10000"
redis-cli CONFIG SET stop-writes-on-bgsave-error yes
redis-cli CONFIG SET rdbcompression yes
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
redis-cli CONFIG SET auto-aof-rewrite-percentage 100
redis-cli CONFIG SET auto-aof-rewrite-min-size 64mb
redis-cli CONFIG SET aof-use-rdb-preamble yes

# Save to redis.conf
redis-cli CONFIG REWRITE
```

## Summary

Redis persistence failures fall into three categories: fork failures from insufficient memory, disk space exhaustion, and file corruption. The quickest diagnostic is `redis-cli INFO persistence` which shows the status of both RDB and AOF subsystems. For write-blocking failures caused by `stop-writes-on-bgsave-error`, temporarily disabling it buys time to fix the root cause while keeping the application functional. Always fix the underlying resource issue and verify persistence is healthy before re-enabling safety settings.
