# How to Use Redis CLI --rdb for Remote RDB Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, CLI, RDB, Backup, Persistence

Description: Learn how to use redis-cli --rdb to trigger a SYNC and download a live RDB snapshot from a remote Redis server without needing filesystem access.

---

`redis-cli --rdb` uses the Redis replication protocol to create a backup of a remote Redis instance. It acts like a replica, requests a full synchronization, and saves the resulting RDB file locally. No SSH or filesystem access to the server is needed.

## How It Works

When you run `redis-cli --rdb`, it:

1. Connects to the Redis server
2. Sends the `SYNC` command (same as a replica)
3. Receives the RDB dump over the network
4. Saves it to the file you specify

This triggers a `BGSAVE` on the server (or reuses the latest RDB), so it has some performance impact.

## Basic Usage

```bash
redis-cli --rdb /tmp/backup.rdb
```

Output:

```text
SYNC sent to master, writing 9876543 bytes to '/tmp/backup.rdb'
Transfer finished with success.
```

## Connecting to a Remote Server

```bash
redis-cli -h redis.example.com -p 6379 -a mypassword --rdb /tmp/backup-$(date +%Y%m%d-%H%M%S).rdb
```

## Connecting with TLS

```bash
redis-cli -h redis.example.com -p 6380 \
  --tls \
  --cacert /etc/ssl/certs/ca.crt \
  -a mypassword \
  --rdb /tmp/backup.rdb
```

## Scheduled Backups with Cron

```bash
#!/bin/bash
# redis-backup.sh
BACKUP_DIR="/backups/redis"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/dump-$TIMESTAMP.rdb"

mkdir -p "$BACKUP_DIR"

redis-cli -h redis.example.com \
  -p 6379 \
  -a "$REDIS_PASSWORD" \
  --rdb "$BACKUP_FILE"

# Keep only last 7 days
find "$BACKUP_DIR" -name "dump-*.rdb" -mtime +7 -delete

echo "Backup completed: $BACKUP_FILE"
```

Schedule with cron:

```bash
0 2 * * * /usr/local/bin/redis-backup.sh >> /var/log/redis-backup.log 2>&1
```

## Verifying the Backup File

Use `rdb` (rdbtools) to validate and inspect the backup:

```bash
pip install rdbtools
rdb --command justkeys /tmp/backup.rdb | head -20
```

Or check the file header:

```bash
xxd /tmp/backup.rdb | head -2
# 00000000: 5245 4449 5330 3031 31fa 0972 6564 6973  REDIS0011..redis
```

A valid RDB file starts with `REDIS` followed by the version number.

## Restoring from the Backup

To restore:

1. Stop the Redis server
2. Copy the RDB file to the Redis data directory
3. Rename it to `dump.rdb`
4. Start Redis

```bash
sudo systemctl stop redis
sudo cp /tmp/backup.rdb /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis
```

## Limitations

- `--rdb` triggers a `BGSAVE` on the source server, which has CPU and I/O impact
- On Redis with ACLs, the user needs `REPLICATION` privilege or `SYNC` command access
- Large datasets create large RDB files and long transfer times

## Summary

`redis-cli --rdb` is a network-based backup tool that downloads a live RDB snapshot from a remote Redis instance using the replication protocol. It requires no filesystem access to the server, making it ideal for automated offsite backups and disaster recovery preparations.
