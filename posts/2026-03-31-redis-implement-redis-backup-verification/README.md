# How to Implement Redis Backup Verification

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Backup, Disaster Recovery, DevOps, Automation

Description: Learn how to automatically verify Redis backups by checking file integrity, testing restoration to a staging instance, and alerting when backups fail validation.

---

A backup that has never been tested is not a backup - it is a hope. Automated backup verification ensures that your Redis dumps are not only created successfully but can actually be restored when needed.

## What Backup Verification Covers

A complete verification process checks:

1. The backup file exists and is not empty
2. The RDB file passes integrity checks (no corruption)
3. The backup can be loaded by a Redis process
4. The key count after restoration matches expectations
5. Alerts fire when verification fails

## Step 1: Verify RDB File Integrity

Redis ships with `redis-check-rdb` to validate RDB files without restoring them:

```bash
#!/bin/bash
# verify-backup-integrity.sh
BACKUP_FILE="$1"

if [ ! -f "$BACKUP_FILE" ]; then
  echo "ERROR: Backup file not found: $BACKUP_FILE"
  exit 1
fi

SIZE=$(stat -c%s "$BACKUP_FILE")
if [ "$SIZE" -lt 100 ]; then
  echo "ERROR: Backup file too small ($SIZE bytes) - likely corrupted or empty"
  exit 1
fi

redis-check-rdb "$BACKUP_FILE"
EXIT_CODE=$?

if [ "$EXIT_CODE" -eq 0 ]; then
  echo "OK: RDB file integrity check passed"
else
  echo "ERROR: RDB file integrity check failed (exit code $EXIT_CODE)"
  exit 1
fi
```

## Step 2: Restore to a Verification Instance

Run a temporary Redis process using the backup file to confirm it loads:

```bash
#!/bin/bash
# restore-verify.sh
BACKUP_FILE="$1"
VERIFY_DIR="/tmp/redis-verify-$$"
VERIFY_PORT=16379

mkdir -p "$VERIFY_DIR"
cp "$BACKUP_FILE" "$VERIFY_DIR/dump.rdb"

# Start a temporary Redis instance pointing at the backup
redis-server \
  --dir "$VERIFY_DIR" \
  --dbfilename dump.rdb \
  --port "$VERIFY_PORT" \
  --daemonize yes \
  --logfile "$VERIFY_DIR/redis-verify.log"

sleep 3

# Check key count
KEYCOUNT=$(redis-cli -p "$VERIFY_PORT" DBSIZE)
echo "Keys loaded from backup: $KEYCOUNT"

# Spot-check a known key if available
redis-cli -p "$VERIFY_PORT" TYPE "known:key:example"

# Shut down verification instance
redis-cli -p "$VERIFY_PORT" SHUTDOWN NOSAVE 2>/dev/null
rm -rf "$VERIFY_DIR"

if [ "$KEYCOUNT" -gt 0 ]; then
  echo "PASS: Backup verification successful ($KEYCOUNT keys)"
  exit 0
else
  echo "FAIL: No keys loaded - backup may be empty or corrupted"
  exit 1
fi
```

## Step 3: Automate Verification After Each Backup

Tie verification into your backup pipeline:

```bash
#!/bin/bash
# full-backup-pipeline.sh
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/tmp/redis-backup-${TIMESTAMP}.rdb"
BUCKET="s3://my-redis-backups/daily"

# Create backup
redis-cli --rdb "$BACKUP_FILE"

# Verify before uploading
if bash verify-backup-integrity.sh "$BACKUP_FILE" && \
   bash restore-verify.sh "$BACKUP_FILE"; then
  aws s3 cp "$BACKUP_FILE" "${BUCKET}/redis-backup-${TIMESTAMP}.rdb"
  echo "Backup created and verified: redis-backup-${TIMESTAMP}.rdb"
else
  echo "Backup verification FAILED - not uploading"
  # Alert via webhook
  curl -s -X POST "$ONEUPTIME_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d '{"message": "Redis backup verification failed", "severity": "critical"}'
  exit 1
fi

rm "$BACKUP_FILE"
```

## Step 4: Track Verification Metrics

Log verification results for trend analysis:

```bash
# Append to verification log
echo "$(date -Iseconds),$(basename $BACKUP_FILE),$KEYCOUNT,PASS" \
  >> /var/log/redis-backup-verify.log
```

Set up a cron job to alert if no successful verification has occurred in the last 2 hours:

```bash
# /etc/cron.d/redis-backup-verify
0 * * * * root /usr/local/bin/check-backup-freshness.sh
```

```bash
#!/bin/bash
# check-backup-freshness.sh
LAST_SUCCESS=$(grep ",PASS" /var/log/redis-backup-verify.log | tail -1 | cut -d',' -f1)
NOW=$(date +%s)
LAST_TS=$(date -d "$LAST_SUCCESS" +%s 2>/dev/null || echo 0)
AGE_HOURS=$(( (NOW - LAST_TS) / 3600 ))

if [ "$AGE_HOURS" -gt 2 ]; then
  echo "ALERT: No successful Redis backup verification in ${AGE_HOURS} hours"
  exit 1
fi
```

## Summary

Redis backup verification combines RDB integrity checking with actual restoration testing to a temporary instance. Automating this as part of every backup pipeline ensures you only upload verified backups. Alert immediately when verification fails - a corrupted or empty backup discovered during a real disaster is worse than having no backup plan at all.
