# How to Automate ClickHouse Backup Scheduling with Cron

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Backup, Cron, Automation, Operation

Description: Learn how to automate ClickHouse backups using cron jobs with proper error handling, logging, and retention management.

---

Manual backups are error-prone. Automating ClickHouse backups with cron ensures consistent, reliable data protection with minimal operational overhead.

## Basic Backup Script

Create a reusable backup script at `/usr/local/bin/clickhouse-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

DATE=$(date +%Y-%m-%d)
HOUR=$(date +%H)
BACKUP_NAME="production_${DATE}_${HOUR}"
LOG_FILE="/var/log/clickhouse-backup.log"
CLICKHOUSE_CLIENT="clickhouse-client --host localhost --port 9000"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting backup: $BACKUP_NAME"

$CLICKHOUSE_CLIENT --query "
BACKUP DATABASE production
TO Disk('backups', '${BACKUP_NAME}/')
SETTINGS async = false
"

if [ $? -eq 0 ]; then
    log "Backup completed successfully: $BACKUP_NAME"
else
    log "ERROR: Backup failed: $BACKUP_NAME"
    exit 1
fi
```

Make it executable:

```bash
chmod +x /usr/local/bin/clickhouse-backup.sh
```

## Cron Schedule

Edit the crontab for the `clickhouse` user:

```bash
crontab -u clickhouse -e
```

Add these entries for daily full backups and hourly incrementals:

```text
# Daily full backup at 2 AM
0 2 * * * /usr/local/bin/clickhouse-backup.sh >> /var/log/clickhouse-backup.log 2>&1

# Hourly incremental backup
0 * * * * /usr/local/bin/clickhouse-incremental-backup.sh >> /var/log/clickhouse-backup.log 2>&1
```

## Incremental Backup Script

```bash
#!/bin/bash
set -euo pipefail

DATE=$(date +%Y-%m-%d)
HOUR=$(date +%H)
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
BASE_BACKUP="production_${YESTERDAY}_02"
BACKUP_NAME="production_incr_${DATE}_${HOUR}"

clickhouse-client --query "
BACKUP DATABASE production
TO Disk('backups', '${BACKUP_NAME}/')
SETTINGS
    async = false,
    base_backup = Disk('backups', '${BASE_BACKUP}/')
"
```

## Retention Management

Clean up old backups automatically:

```bash
#!/bin/bash
# Keep last 7 daily backups and last 24 hourly incrementals
BACKUP_DIR="/mnt/backups"
KEEP_DAILY=7
KEEP_HOURLY=24

# Remove old daily backups
ls -dt "$BACKUP_DIR"/production_20*_02/ 2>/dev/null | tail -n "+$((KEEP_DAILY + 1))" | xargs -r rm -rf

# Remove old hourly incrementals
ls -dt "$BACKUP_DIR"/production_incr_* 2>/dev/null | tail -n "+$((KEEP_HOURLY + 1))" | xargs -r rm -rf

echo "Retention cleanup complete"
```

## Sending Backup Alerts

Add alerting to the backup script using curl:

```bash
send_alert() {
    local message="$1"
    curl -s -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-type: application/json' \
        --data "{\"text\": \"ClickHouse Backup: $message\"}"
}

# In backup script, call on failure:
send_alert "FAILED: $BACKUP_NAME on $(hostname)"
```

## Verifying Backup Success with system.backups

After each backup, verify it completed successfully:

```bash
STATUS=$(clickhouse-client --query "
SELECT status
FROM system.backups
WHERE name LIKE '%${BACKUP_NAME}%'
ORDER BY start_time DESC
LIMIT 1
FORMAT TSV")

if [ "$STATUS" != "BACKUP_CREATED" ]; then
    echo "Backup verification failed: status=$STATUS"
    exit 1
fi
```

## Summary

Automating ClickHouse backups with cron requires a solid shell script with error handling, logging, and retention cleanup. Use a daily full backup at off-peak hours, incremental backups throughout the day, and a separate retention script to manage disk space. Always verify backup status via `system.backups` and alert on failures.
