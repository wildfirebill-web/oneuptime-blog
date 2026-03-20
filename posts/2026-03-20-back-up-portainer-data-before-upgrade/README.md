# How to Back Up Portainer Data Before an Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Backup, Docker, Maintenance, DevOps

Description: Learn multiple methods to back up Portainer data before performing an upgrade to protect against data loss or failed migrations.

---

Portainer stores all its configuration in a BoltDB database inside the data volume. Before any upgrade, creating a reliable backup ensures you can quickly restore if something goes wrong.

## What Gets Backed Up

The Portainer data volume (`portainer_data`) contains:
- Database (`portainer.db`) — all settings, users, environments, stacks
- TLS certificates
- Compose files for managed stacks
- Custom templates
- Edge agent configurations

## Method 1: Volume Tar Archive (Recommended)

The simplest and most reliable backup method:

```bash
# Stop Portainer for a consistent backup (recommended)
docker stop portainer

# Create a timestamped tar archive of the entire data volume
docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)/backups":/backup \
  alpine \
  tar czf /backup/portainer_backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .

# Restart Portainer after backup
docker start portainer

echo "Backup saved to: backups/portainer_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
```

## Method 2: Copy Only the Database File

If you only need to back up the configuration database:

```bash
# Stop Portainer for consistent file copy
docker stop portainer

# Copy just the BoltDB database file
docker run --rm \
  -v portainer_data:/data \
  -v "$(pwd)/backups":/backup \
  alpine \
  cp /data/portainer.db /backup/portainer_$(date +%Y%m%d_%H%M%S).db

docker start portainer
```

## Method 3: Portainer Business Edition API Backup

Portainer BE includes a dedicated backup API endpoint:

```bash
# Trigger a backup via the Portainer BE API
# Replace <TOKEN> with your JWT token from login
curl -X POST \
  https://localhost:9443/api/backup \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"password": "optionalEncryptionPassword"}' \
  --output portainer_backup_$(date +%Y%m%d).tar.gz \
  --insecure

echo "API backup downloaded"
```

## Method 4: Automated Backup Script

Set up a cron job for regular automated backups:

```bash
#!/bin/bash
# /usr/local/bin/portainer-backup.sh
# Run daily via: crontab -e -> 0 2 * * * /usr/local/bin/portainer-backup.sh

BACKUP_DIR="/opt/backups/portainer"
RETENTION_DAYS=7

mkdir -p "$BACKUP_DIR"

# Stop for consistent backup
docker stop portainer

# Create backup
docker run --rm \
  -v portainer_data:/data \
  -v "$BACKUP_DIR":/backup \
  alpine \
  tar czf "/backup/portainer_$(date +%Y%m%d_%H%M%S).tar.gz" -C /data .

# Restart immediately
docker start portainer

# Remove backups older than retention period
find "$BACKUP_DIR" -name "portainer_*.tar.gz" -mtime +"$RETENTION_DAYS" -delete

echo "Backup complete. Current backups:"
ls -lh "$BACKUP_DIR"
```

## Verify Your Backup

Always verify the backup is valid before proceeding with the upgrade:

```bash
# List contents of the backup archive to verify integrity
tar tzf backups/portainer_backup_latest.tar.gz | head -20

# Check the archive is not corrupted
tar tzf backups/portainer_backup_latest.tar.gz > /dev/null && echo "Backup OK" || echo "Backup CORRUPTED"
```

---

*Set up automated backup monitoring and alerts with [OneUptime](https://oneuptime.com) to ensure your backup jobs succeed.*
