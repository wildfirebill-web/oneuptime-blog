# How to Set Up Redis Automated Backup to Azure Blob Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Azure, Backup, Disaster Recovery, DevOps

Description: Learn how to automate Redis RDB backups to Azure Blob Storage with scheduled uploads, retention policies, and monitoring for backup failures.

---

Azure Blob Storage provides durable, geo-redundant storage for Redis backup files at low cost. Combined with a cron job or systemd timer, you can ensure Redis backups run automatically and are retained reliably.

## Prerequisites

- Redis running on a Linux VM or container with access to Azure
- Azure CLI installed and authenticated (`az login`)
- An Azure Storage Account and Blob container

## Step 1: Create Azure Blob Storage Container

```bash
# Create a storage account (if not already exists)
az storage account create \
  --name myredisbackups \
  --resource-group my-rg \
  --location eastus \
  --sku Standard_LRS

# Create a container for Redis backups
az storage container create \
  --name redis-backups \
  --account-name myredisbackups \
  --public-access off
```

## Step 2: Create the Backup Script

```bash
#!/bin/bash
# /usr/local/bin/redis-backup-azure.sh

set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/tmp/redis-backup-${TIMESTAMP}.rdb"
STORAGE_ACCOUNT="myredisbackups"
CONTAINER="redis-backups"
REDIS_HOST="${REDIS_HOST:-127.0.0.1}"
REDIS_PORT="${REDIS_PORT:-6379}"
RETENTION_DAYS="${RETENTION_DAYS:-30}"

# Create RDB snapshot
echo "[$(date -Iseconds)] Creating Redis backup..."
redis-cli -h "$REDIS_HOST" -p "$REDIS_PORT" --rdb "$BACKUP_FILE"

BACKUP_SIZE=$(stat -c%s "$BACKUP_FILE")
echo "[$(date -Iseconds)] Backup created: ${BACKUP_SIZE} bytes"

# Verify backup integrity
redis-check-rdb "$BACKUP_FILE" || {
  echo "ERROR: RDB integrity check failed"
  rm -f "$BACKUP_FILE"
  exit 1
}

# Upload to Azure Blob Storage
az storage blob upload \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER" \
  --file "$BACKUP_FILE" \
  --name "redis-backup-${TIMESTAMP}.rdb" \
  --auth-mode login

echo "[$(date -Iseconds)] Uploaded redis-backup-${TIMESTAMP}.rdb to Azure"

# Clean up local file
rm -f "$BACKUP_FILE"

# Enforce retention policy - delete blobs older than RETENTION_DAYS
CUTOFF_DATE=$(date -d "${RETENTION_DAYS} days ago" +%Y-%m-%dT%H:%M:%SZ)
az storage blob list \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER" \
  --auth-mode login \
  --query "[?properties.lastModified < '${CUTOFF_DATE}'].name" \
  --output tsv | while read -r blob_name; do
    az storage blob delete \
      --account-name "$STORAGE_ACCOUNT" \
      --container-name "$CONTAINER" \
      --name "$blob_name" \
      --auth-mode login
    echo "Deleted old backup: $blob_name"
  done

echo "[$(date -Iseconds)] Backup pipeline complete"
```

Make the script executable:

```bash
chmod +x /usr/local/bin/redis-backup-azure.sh
```

## Step 3: Schedule with Cron or Systemd

Using cron (every 15 minutes):

```bash
# Add to /etc/cron.d/redis-backup
*/15 * * * * redis /usr/local/bin/redis-backup-azure.sh >> /var/log/redis-backup.log 2>&1
```

Using a systemd timer for more reliable scheduling:

```text
# /etc/systemd/system/redis-backup.service
[Unit]
Description=Redis Backup to Azure

[Service]
Type=oneshot
User=redis
ExecStart=/usr/local/bin/redis-backup-azure.sh
StandardOutput=journal
StandardError=journal
```

```text
# /etc/systemd/system/redis-backup.timer
[Unit]
Description=Run Redis backup every 15 minutes

[Timer]
OnCalendar=*:0/15
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now redis-backup.timer
systemctl status redis-backup.timer
```

## Step 4: Restore from Azure Backup

```bash
#!/bin/bash
# List available backups
az storage blob list \
  --account-name myredisbackups \
  --container-name redis-backups \
  --auth-mode login \
  --query "[].{Name:name, LastModified:properties.lastModified}" \
  --output table

# Download the desired backup
az storage blob download \
  --account-name myredisbackups \
  --container-name redis-backups \
  --name "redis-backup-20260331_120000.rdb" \
  --file /tmp/restore.rdb \
  --auth-mode login

# Restore
systemctl stop redis
cp /tmp/restore.rdb /var/lib/redis/dump.rdb
chown redis:redis /var/lib/redis/dump.rdb
systemctl start redis
redis-cli DBSIZE
```

## Summary

Automating Redis backups to Azure Blob Storage requires a shell script that creates an RDB snapshot, verifies its integrity, uploads it via Azure CLI, and enforces retention. Scheduling with systemd timers is more reliable than cron for ensuring backups run even after reboots. Monitor backup logs in OneUptime to alert when the pipeline fails.
