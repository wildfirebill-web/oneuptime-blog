# How to Back Up Portainer Database Before Major Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Backup, Safety, Upgrade, Administration, BoltDB

Description: Learn how to create a point-in-time backup of the Portainer database before performing major changes like upgrades, bulk deletions, or configuration overhauls.

---

Taking a backup before any major change to Portainer gives you a fast rollback path if something goes wrong. This should be a standard step before upgrades, bulk stack changes, or team restructuring.

## Quick Pre-Change Backup (30 Seconds)

```bash
#!/bin/bash
# Quick backup before any major change

BACKUP_FILE="/tmp/portainer-pre-change-$(date +%Y%m%d-%H%M%S).tar.gz"

# Create backup without stopping Portainer (uses live database)
# For maximum safety, stop Portainer first
docker run --rm \
  -v portainer_data:/data \
  alpine \
  tar czpf - /data > "$BACKUP_FILE"

echo "Backup saved to: $BACKUP_FILE"
echo "Size: $(du -sh $BACKUP_FILE | cut -f1)"
echo "To restore: see portainer-restore guide"
```

## Before Upgrading Portainer

Always backup before upgrading, as Portainer runs database migrations that cannot always be reversed:

```bash
# 1. Backup
docker run --rm -v portainer_data:/data alpine tar czpf - /data > \
  portainer-pre-upgrade-$(portainer --version 2>/dev/null || echo "unknown").tar.gz

# 2. Pull new version
docker pull portainer/portainer-ce:latest

# 3. Upgrade
docker stop portainer && docker rm portainer
docker run -d --name portainer portainer/portainer-ce:latest \
  -v portainer_data:/data ...
```

## Before Bulk Stack Operations

When deleting or modifying many stacks at once:

```bash
# Backup before bulk delete
docker run --rm -v portainer_data:/data alpine tar czpf - /data > \
  /tmp/portainer-pre-bulkdelete.tar.gz

# Proceed with the bulk operation in Portainer UI
```

## Verifying the Backup Before Proceeding

Always verify the backup is complete and readable before making changes:

```bash
BACKUP_FILE="/tmp/portainer-pre-change-20260320.tar.gz"

# Check the file is not corrupted
tar tzf "$BACKUP_FILE" > /dev/null && echo "Backup OK" || echo "BACKUP CORRUPTED - do not proceed"

# Verify the database file is present
tar tzf "$BACKUP_FILE" | grep "portainer.db" || echo "WARNING: portainer.db not in backup"

# Check file size is non-zero
[ -s "$BACKUP_FILE" ] && echo "Backup has content" || echo "BACKUP IS EMPTY"
```

## Rolling Back if Something Goes Wrong

```bash
# If the major change caused problems:
# 1. Stop Portainer
docker stop portainer && docker rm portainer

# 2. Remove the (now broken) data volume
docker volume rm portainer_data

# 3. Restore from backup
docker volume create portainer_data
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine tar xzpf /backup/portainer-pre-change-20260320.tar.gz -C /

# 4. Start the previous Portainer version
docker run -d --name portainer -v portainer_data:/data portainer/portainer-ce:previous-tag
```
