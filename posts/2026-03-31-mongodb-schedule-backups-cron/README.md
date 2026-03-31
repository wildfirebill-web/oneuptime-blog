# How to Schedule MongoDB Backups with Cron

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Cron, Automation, Database

Description: Learn how to automate MongoDB backups using cron jobs with mongodump, covering scheduling, compression, retention policies, and alerting.

---

Automating MongoDB backups with cron ensures you always have a recent recovery point without manual intervention. This guide walks through setting up reliable scheduled backups on Linux.

## Creating a Backup Script

Start with a reusable shell script:

```bash
#!/bin/bash
# /usr/local/bin/mongodb-backup.sh

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_DIR="/var/backups/mongodb"
MONGO_URI="mongodb://backupuser:password@localhost:27017"
RETENTION_DAYS=7

mkdir -p "$BACKUP_DIR"

# Run mongodump and compress output
mongodump \
  --uri="$MONGO_URI" \
  --authenticationDatabase=admin \
  --gzip \
  --archive="$BACKUP_DIR/backup_$TIMESTAMP.gz"

if [ $? -eq 0 ]; then
  echo "Backup succeeded: backup_$TIMESTAMP.gz"
else
  echo "Backup FAILED at $TIMESTAMP" >&2
  exit 1
fi

# Remove backups older than retention period
find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete
echo "Old backups cleaned up"
```

Make it executable:

```bash
chmod +x /usr/local/bin/mongodb-backup.sh
```

## Creating a Dedicated Backup User

Limit permissions for the backup account:

```javascript
use admin
db.createUser({
  user: "backupuser",
  pwd: "strongpassword",
  roles: [{ role: "backup", db: "admin" }]
})
```

## Scheduling with Cron

Open the crontab editor:

```bash
crontab -e
```

Add entries for different schedules:

```text
# Daily backup at 2:00 AM
0 2 * * * /usr/local/bin/mongodb-backup.sh >> /var/log/mongodb-backup.log 2>&1

# Hourly backup during business hours (8 AM - 6 PM, Monday-Friday)
0 8-18 * * 1-5 /usr/local/bin/mongodb-backup.sh >> /var/log/mongodb-backup.log 2>&1

# Weekly full backup on Sunday at 1:00 AM
0 1 * * 0 /usr/local/bin/mongodb-backup.sh >> /var/log/mongodb-backup-weekly.log 2>&1
```

## Verifying Backups

Add a verification step to your script to confirm the archive is valid:

```bash
# Verify the backup can be read
mongorestore \
  --uri="mongodb://localhost:27017" \
  --gzip \
  --archive="$BACKUP_DIR/backup_$TIMESTAMP.gz" \
  --dryRun \
  --quiet

if [ $? -ne 0 ]; then
  echo "Backup verification FAILED" >&2
fi
```

## Uploading to Remote Storage

Push backups to S3 for offsite retention:

```bash
# Install awscli, then add to script:
aws s3 cp "$BACKUP_DIR/backup_$TIMESTAMP.gz" \
  "s3://my-mongo-backups/daily/backup_$TIMESTAMP.gz"
```

## Monitoring Backup Health with OneUptime

Track backup success rates using OneUptime heartbeats. After a successful backup run, ping a heartbeat URL:

```bash
# At the end of the backup script
curl -s "https://oneuptime.com/api/heartbeat/YOUR_HEARTBEAT_KEY" > /dev/null
```

If the heartbeat is missed, OneUptime fires an alert, notifying your team that a scheduled backup did not run.

## Summary

Scheduling MongoDB backups with cron requires a reliable script, a least-privilege backup user, and a retention policy. Pairing cron-based backups with S3 offsite storage and heartbeat monitoring gives you confidence that backups are running and valid without manual checks.

