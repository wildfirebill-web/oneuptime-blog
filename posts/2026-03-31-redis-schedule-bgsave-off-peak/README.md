# How to Schedule Redis BGSAVE at Off-Peak Hours

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, BGSAVE, Persistence, Performance

Description: Schedule Redis BGSAVE snapshots during off-peak hours using cron and CONFIG SET to minimize the performance impact of fork and copy-on-write on production traffic.

---

Redis BGSAVE forks the server process, which can cause a brief latency spike. During the save, copy-on-write overhead increases memory usage proportionally to the write rate. Scheduling saves during off-peak hours reduces the impact on production traffic.

## Disabling Automatic Saves

First, disable the automatic save triggers so you control the timing:

```bash
redis-cli CONFIG SET save ""
redis-cli CONFIG GET save
# Should return empty string
```

Or in `redis.conf`:

```text
save ""
```

## Scheduling via Cron

Create a cron job that triggers BGSAVE at a specific time:

```bash
crontab -e
```

```text
# Redis BGSAVE at 2 AM daily
0 2 * * * redis-cli BGSAVE

# Redis BGSAVE every 4 hours during low-traffic windows
0 2,6,10 * * * redis-cli BGSAVE
```

If Redis requires authentication:

```text
0 2 * * * redis-cli -a yourpassword BGSAVE
```

## Waiting for BGSAVE Completion

For scripts that need to confirm the save completed:

```bash
#!/bin/bash
# Trigger BGSAVE and wait for completion

redis-cli BGSAVE

echo "Waiting for BGSAVE to complete..."
while true; do
    STATUS=$(redis-cli INFO persistence | grep rdb_current_bgsave_time_sec | cut -d: -f2 | tr -d '\r')
    if [ "$STATUS" = "-1" ]; then
        echo "BGSAVE complete"
        break
    fi
    echo "BGSAVE in progress (${STATUS}s)..."
    sleep 2
done

# Check result
RESULT=$(redis-cli INFO persistence | grep rdb_last_bgsave_status | cut -d: -f2 | tr -d '\r ')
if [ "$RESULT" = "ok" ]; then
    echo "BGSAVE succeeded"
else
    echo "BGSAVE failed: $RESULT"
    exit 1
fi
```

## Scheduled Backup with File Rotation

Combine BGSAVE with timestamped backups:

```bash
#!/bin/bash
# /usr/local/bin/redis-backup.sh

REDIS_DIR=$(redis-cli CONFIG GET dir | tail -1)
REDIS_FILE=$(redis-cli CONFIG GET dbfilename | tail -1)
BACKUP_DIR=/backup/redis
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/dump-${DATE}.rdb"

# Ensure backup directory exists
mkdir -p "$BACKUP_DIR"

# Trigger BGSAVE
redis-cli BGSAVE

# Wait for completion
while true; do
    STATUS=$(redis-cli INFO persistence | grep rdb_current_bgsave_time_sec | cut -d: -f2 | tr -d '\r')
    [ "$STATUS" = "-1" ] && break
    sleep 2
done

# Copy the RDB file
cp "${REDIS_DIR}/${REDIS_FILE}" "$BACKUP_FILE"

# Validate the backup
if redis-check-rdb "$BACKUP_FILE" > /dev/null 2>&1; then
    echo "Backup saved: $BACKUP_FILE"
else
    echo "ALERT: Backup validation failed: $BACKUP_FILE"
fi

# Keep only the last 7 days of backups
find "$BACKUP_DIR" -name "dump-*.rdb" -mtime +7 -delete
```

Schedule it:

```bash
chmod +x /usr/local/bin/redis-backup.sh
echo "0 2 * * * root /usr/local/bin/redis-backup.sh >> /var/log/redis-backup.log 2>&1" | \
  sudo tee /etc/cron.d/redis-backup
```

## Monitoring Save Timing

Track when saves actually ran to confirm the schedule is working:

```bash
redis-cli LASTSAVE
# Returns Unix timestamp
date -d @$(redis-cli LASTSAVE)
```

Or check in logs:

```bash
grep "Background saving" /var/log/redis/redis-server.log | tail -10
```

## Using Redis on Kubernetes

In Kubernetes, use a CronJob instead of cron:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-bgsave
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: redis-backup
            image: redis:7
            command:
            - sh
            - -c
            - redis-cli -h redis-service BGSAVE && sleep 5 && redis-cli -h redis-service LASTSAVE
          restartPolicy: OnFailure
```

## Summary

Schedule Redis BGSAVE at off-peak hours by disabling automatic save triggers with `CONFIG SET save ""` and using cron or Kubernetes CronJobs to trigger `BGSAVE` at predictable low-traffic windows. Use a wrapper script that waits for save completion, validates the output file, and rotates old backups to maintain a rolling archive. Monitor `LASTSAVE` and Redis logs to confirm the schedule is executing reliably.
