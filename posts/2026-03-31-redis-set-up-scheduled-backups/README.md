# How to Set Up Redis Scheduled Backups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Backup, Cron, RDB, AOF, Data Protection, System Administration

Description: Set up scheduled Redis backups using cron jobs, RDB snapshots, and AOF persistence, with automated upload to S3 or remote storage for reliable data protection.

---

## Redis Persistence Options

Redis supports two persistence mechanisms:
- **RDB** - point-in-time snapshots of the dataset; compact, fast restore
- **AOF** - append-only log of every write command; more durable, larger files

For scheduled backups, you typically trigger RDB snapshots and copy the resulting `dump.rdb` file.

## Configure RDB Snapshots

Edit `/etc/redis/redis.conf`:

```text
# Save every 900s if at least 1 key changed
save 900 1
# Save every 300s if at least 10 keys changed
save 300 10
# Save every 60s if at least 10000 keys changed
save 60 10000

# RDB file location
dir /var/lib/redis
dbfilename dump.rdb

# Compress RDB with LZF
rdbcompression yes
```

## Manual Backup Trigger

```bash
# Trigger a background save
redis-cli BGSAVE

# Wait for it to complete
redis-cli LASTSAVE

# Copy the RDB file
cp /var/lib/redis/dump.rdb /backups/redis/dump-$(date +%Y%m%d-%H%M%S).rdb
```

## Backup Script

```bash
#!/bin/bash
set -e

REDIS_HOST="127.0.0.1"
REDIS_PORT="6379"
REDIS_AUTH=""
BACKUP_DIR="/backups/redis"
REDIS_DATA_DIR="/var/lib/redis"
RETENTION_DAYS=7

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/dump-$DATE.rdb"

mkdir -p "$BACKUP_DIR"

# Trigger background save
if [ -n "$REDIS_AUTH" ]; then
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" -a "$REDIS_AUTH" BGSAVE
else
    redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" BGSAVE
fi

# Wait for save to complete
echo "Waiting for BGSAVE to complete..."
LASTSAVE_BEFORE=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" LASTSAVE)

while true; do
    sleep 2
    LASTSAVE_AFTER=$(redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" LASTSAVE)
    if [ "$LASTSAVE_AFTER" != "$LASTSAVE_BEFORE" ]; then
        break
    fi
    # Check bgsave_in_progress
    if redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" INFO persistence | grep -q "rdb_bgsave_in_progress:0"; then
        break
    fi
done

# Copy the RDB file
cp "$REDIS_DATA_DIR/dump.rdb" "$BACKUP_FILE"
echo "Backup saved to: $BACKUP_FILE"

# Remove old backups
find "$BACKUP_DIR" -name "dump-*.rdb" -mtime +$RETENTION_DAYS -delete
echo "Cleaned up backups older than $RETENTION_DAYS days"
```

## Schedule with Cron

```bash
# Edit crontab
crontab -e
```

Add backup schedule:

```text
# Run Redis backup every hour
0 * * * * /usr/local/bin/redis-backup.sh >> /var/log/redis-backup.log 2>&1

# Run daily full backup at 2 AM
0 2 * * * /usr/local/bin/redis-backup.sh >> /var/log/redis-backup.log 2>&1
```

## Upload Backups to Amazon S3

```bash
#!/bin/bash
# redis-backup-s3.sh
S3_BUCKET="s3://my-redis-backups"
BACKUP_DIR="/backups/redis"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/dump-$DATE.rdb"

# ... (run BGSAVE and copy dump.rdb as above) ...

# Upload to S3
aws s3 cp "$BACKUP_FILE" "$S3_BUCKET/$(hostname)/$DATE/dump.rdb" \
    --storage-class STANDARD_IA \
    --sse AES256

echo "Uploaded to S3: $S3_BUCKET/$(hostname)/$DATE/dump.rdb"

# Delete local file after upload (optional)
rm "$BACKUP_FILE"
```

## Backup AOF File

```bash
#!/bin/bash
# Backup the AOF file
AOF_FILE="/var/lib/redis/appendonly.aof"
BACKUP_DIR="/backups/redis"
DATE=$(date +%Y%m%d-%H%M%S)

# Rewrite AOF first to compact it
redis-cli BGREWRITEAOF
sleep 5

cp "$AOF_FILE" "$BACKUP_DIR/appendonly-$DATE.aof"
gzip "$BACKUP_DIR/appendonly-$DATE.aof"
echo "AOF backup created: $BACKUP_DIR/appendonly-$DATE.aof.gz"
```

## Verify Backup Integrity

```bash
# Check RDB file validity
redis-check-rdb /backups/redis/dump-20240401-020000.rdb

# Check AOF file validity
redis-check-aof /backups/redis/appendonly-20240401-020000.aof
```

## Restore from Backup

```bash
# Stop Redis
sudo systemctl stop redis

# Copy backup over the active RDB
cp /backups/redis/dump-20240401-020000.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb

# Start Redis (it loads the RDB on startup)
sudo systemctl start redis

# Verify
redis-cli DBSIZE
```

## Summary

Scheduled Redis backups use BGSAVE to trigger non-blocking RDB snapshots, which are then copied to a backup directory and optionally uploaded to S3. Cron schedules the backup script hourly or daily depending on your RPO requirements. Always verify backup integrity with `redis-check-rdb` and test restore procedures periodically to confirm backups are usable.
