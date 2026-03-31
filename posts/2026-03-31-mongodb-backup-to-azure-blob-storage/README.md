# How to Back Up MongoDB to Azure Blob Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Backup, Azure, Storage, Automation

Description: Learn how to automate MongoDB backups to Azure Blob Storage using mongodump and the Azure CLI, with lifecycle management for retention.

---

## Why Use Azure Blob Storage for MongoDB Backups

Azure Blob Storage offers highly durable geo-redundant storage (GRS) with 16 nines of durability in paired regions. For organizations already running workloads on Azure, it integrates naturally with Managed Identity authentication, Azure Monitor for backup job alerting, and Azure Blob lifecycle management for automated tiering and deletion.

## Prerequisites

- `mongodump` installed and accessible
- Azure CLI (`az`) installed and authenticated
- An Azure Storage account and container
- Azure role assignment: Storage Blob Data Contributor on the container

## Backup Script

```bash
#!/bin/bash
# mongodb-backup-azure.sh

set -euo pipefail

MONGO_URI="${MONGO_URI:-mongodb://user:pass@localhost:27017}"
STORAGE_ACCOUNT="${STORAGE_ACCOUNT:-mymongobackups}"
CONTAINER="${CONTAINER:-mongodb-backups}"
BACKUP_TMP="/tmp/mongodb-backup"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
ARCHIVE_NAME="mongodb-${TIMESTAMP}.archive.gz"

echo "Starting MongoDB backup: $TIMESTAMP"

mkdir -p "$BACKUP_TMP"

# Create compressed archive
mongodump \
  --uri "$MONGO_URI" \
  --gzip \
  --archive="$BACKUP_TMP/$ARCHIVE_NAME"

BACKUP_SIZE=$(du -sh "$BACKUP_TMP/$ARCHIVE_NAME" | cut -f1)
echo "Backup size: $BACKUP_SIZE"

# Upload to Azure Blob Storage
echo "Uploading to Azure Blob Storage..."
az storage blob upload \
  --account-name "$STORAGE_ACCOUNT" \
  --container-name "$CONTAINER" \
  --name "backups/$ARCHIVE_NAME" \
  --file "$BACKUP_TMP/$ARCHIVE_NAME" \
  --auth-mode login \
  --tier Cool

echo "Upload complete: $ARCHIVE_NAME"

# Cleanup local temp file
rm -f "$BACKUP_TMP/$ARCHIVE_NAME"
echo "Backup complete"
```

## Authenticating with Managed Identity

For Azure VMs, avoid storing credentials by using a system-assigned Managed Identity:

```bash
# Assign Storage Blob Data Contributor to the VM's managed identity
az role assignment create \
  --assignee <vm-principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/mymongobackups/blobServices/default/containers/mongodb-backups"
```

In the backup script, use `--auth-mode login` with the Azure CLI which automatically uses the VM's identity token.

## Lifecycle Management Policy

Create a lifecycle policy to tier backups to archive and then delete old ones:

```json
{
  "rules": [
    {
      "name": "mongodb-backup-lifecycle",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["backups/mongodb-"]
        },
        "actions": {
          "baseBlob": {
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 30
            },
            "delete": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        }
      }
    }
  ]
}
```

Apply it:

```bash
az storage account management-policy create \
  --account-name mymongobackups \
  --resource-group myResourceGroup \
  --policy @lifecycle-policy.json
```

## Scheduling with Azure Automation

For fully managed scheduling, use an Azure Automation runbook instead of cron:

```python
import subprocess
import datetime
import os

def main():
    timestamp = datetime.datetime.utcnow().strftime('%Y%m%d-%H%M%S')
    archive = f"/tmp/mongodb-{timestamp}.archive.gz"

    # Run mongodump
    result = subprocess.run([
        "mongodump",
        "--uri", os.environ["MONGO_URI"],
        "--gzip",
        f"--archive={archive}"
    ], capture_output=True, text=True)

    if result.returncode != 0:
        raise Exception(f"mongodump failed: {result.stderr}")

    # Upload to blob
    subprocess.run([
        "az", "storage", "blob", "upload",
        "--account-name", "mymongobackups",
        "--container-name", "mongodb-backups",
        "--name", f"backups/mongodb-{timestamp}.archive.gz",
        "--file", archive,
        "--auth-mode", "login"
    ], check=True)

    os.remove(archive)
    print(f"Backup completed: mongodb-{timestamp}.archive.gz")
```

## Restoring from Azure Blob

```bash
# Download the backup
az storage blob download \
  --account-name mymongobackups \
  --container-name mongodb-backups \
  --name "backups/mongodb-20240115-020000.archive.gz" \
  --file /tmp/restore.archive.gz \
  --auth-mode login

# Restore to MongoDB
mongorestore \
  --uri "mongodb://user:pass@localhost:27017" \
  --gzip \
  --archive=/tmp/restore.archive.gz \
  --drop
```

## Summary

Backing up MongoDB to Azure Blob Storage is straightforward with `mongodump` and the Azure CLI. Use Managed Identity for credential-free authentication on Azure VMs, blob lifecycle policies to automate tiering and deletion, and Azure Automation or cron for scheduling. Monitor backup job health with OneUptime heartbeat monitors to ensure continuous coverage across your backup window.
