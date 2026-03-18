# How to Backup Podman Volumes

Author: [nawazdhandala](https://github.com/nawazdhandala)

Tags: Podman, Volumes, Backup, Data Persistence, DevOps

Description: A step-by-step guide to backing up Podman volumes, including named volumes, bind mounts, and strategies for consistent data snapshots.

---

> Your containers are replaceable. Your data is not. Learn how to back up Podman volumes so that database records, uploads, and configuration files survive any failure.

Volumes are where the important data lives. While container filesystems are ephemeral and can be rebuilt from images, volumes hold databases, user uploads, configuration state, and everything else that cannot be regenerated from code. Losing a volume means losing data, and losing data means losing trust. This guide covers practical methods for backing up Podman volumes reliably.

---

## Types of Podman Storage

Podman supports two primary mechanisms for persistent storage:

**Named volumes** are managed by Podman and stored in a location Podman controls (typically under `~/.local/share/containers/storage/volumes/` for rootless or `/var/lib/containers/storage/volumes/` for rootful).

**Bind mounts** map a directory on the host directly into the container. The data lives on the host filesystem and is accessible outside the container.

The backup approach differs for each type.

## Identifying Volumes to Back Up

Start by listing all volumes on the system:

```bash
podman volume ls
```

To see which containers use which volumes:

```bash
podman ps -a --format "{{.Names}}" | while read container; do
    echo "=== $container ==="
    podman inspect "$container" --format '{{range .Mounts}}Type:{{.Type}} Source:{{.Source}} Dest:{{.Destination}}{{println}}{{end}}'
done
```

This shows every mount for every container, making it clear which volumes contain critical data.

## Backing Up Named Volumes

Named volumes cannot be directly copied because Podman manages their storage location. The standard approach is to mount the volume into a temporary container and use `tar` to create an archive:

```bash
podman run --rm \
    -v my-database-volume:/source:ro \
    -v /backups:/backup \
    alpine tar czf /backup/my-database-volume-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
```

This command does the following:

1. Creates a temporary Alpine container (small and fast).
2. Mounts the target volume at `/source` in read-only mode.
3. Mounts the backup directory at `/backup`.
4. Creates a compressed tar archive of the volume contents.
5. Removes the container when done (`--rm`).

The read-only mount (`:ro`) prevents accidental modifications to the volume during backup.

## Backing Up Bind Mounts

Bind mounts are simpler because the data is directly accessible on the host. You can use standard filesystem tools:

```bash
tar czf /backups/app-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /data/my-app .
```

Or use `rsync` for incremental backups:

```bash
rsync -avz /data/my-app/ /backups/my-app-latest/
```

## Ensuring Data Consistency

Backing up a volume while a container is actively writing to it can produce an inconsistent snapshot. A database backed up mid-transaction may be corrupted. There are several approaches to handle this:

### Approach 1: Stop the Container

The safest option is to stop the container before backing up:

```bash
podman stop postgres-db
podman run --rm \
    -v postgres-data:/source:ro \
    -v /backups:/backup \
    alpine tar czf /backup/postgres-data-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
podman start postgres-db
```

This guarantees consistency but causes downtime.

### Approach 2: Use Application-Level Backup Tools

For databases, use the database's own dump utility instead of copying raw files:

```bash
# PostgreSQL
podman exec postgres-db pg_dump -U postgres mydb > /backups/mydb-$(date +%Y%m%d-%H%M%S).sql

# MySQL/MariaDB
podman exec mysql-db mysqldump -u root -p"$MYSQL_ROOT_PASSWORD" --all-databases > /backups/all-databases-$(date +%Y%m%d-%H%M%S).sql

# MongoDB
podman exec mongo-db mongodump --archive=/tmp/backup.archive
podman cp mongo-db:/tmp/backup.archive /backups/mongo-$(date +%Y%m%d-%H%M%S).archive

# Redis
podman exec redis-cache redis-cli BGSAVE
sleep 2
podman cp redis-cache:/data/dump.rdb /backups/redis-$(date +%Y%m%d-%H%M%S).rdb
```

Application-level backups produce consistent snapshots without stopping the container.

### Approach 3: Pause the Container

If stopping is not an option and the application does not have its own backup tool, pausing the container freezes all processes without terminating them:

```bash
podman pause my-container
podman run --rm \
    -v my-volume:/source:ro \
    -v /backups:/backup \
    alpine tar czf /backup/my-volume-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
podman unpause my-container
```

Pausing is faster than stop/start but still causes a brief service interruption.

## A Complete Volume Backup Script

Here is a script that backs up all named volumes on a system:

```bash
#!/bin/bash

BACKUP_DIR="/backups/podman-volumes/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"

LOG_FILE="$BACKUP_DIR/backup.log"

echo "Starting volume backup at $(date)" | tee "$LOG_FILE"

for volume in $(podman volume ls -q); do
    echo "Backing up volume: $volume" | tee -a "$LOG_FILE"

    # Get volume size estimate
    SIZE=$(podman volume inspect "$volume" --format '{{.Mountpoint}}')

    podman run --rm \
        -v "${volume}:/source:ro" \
        -v "$BACKUP_DIR:/backup" \
        alpine tar czf "/backup/${volume}.tar.gz" -C /source . 2>> "$LOG_FILE"

    if [ $? -eq 0 ]; then
        BACKUP_SIZE=$(du -h "$BACKUP_DIR/${volume}.tar.gz" | cut -f1)
        echo "  Success: ${volume}.tar.gz ($BACKUP_SIZE)" | tee -a "$LOG_FILE"
    else
        echo "  FAILED: $volume" | tee -a "$LOG_FILE"
    fi
done

echo "Backup complete at $(date)" | tee -a "$LOG_FILE"
echo "Location: $BACKUP_DIR"
```

## Restoring Volumes from Backup

To restore a named volume from a tar archive:

```bash
# Create a fresh volume
podman volume create my-database-volume

# Restore data into it
podman run --rm \
    -v my-database-volume:/target \
    -v /backups:/backup:ro \
    alpine tar xzf /backup/my-database-volume.tar.gz -C /target
```

To restore and verify in one step:

```bash
podman run --rm \
    -v my-database-volume:/target \
    -v /backups:/backup:ro \
    alpine sh -c '
        tar xzf /backup/my-database-volume.tar.gz -C /target &&
        echo "Files restored:" &&
        ls -la /target/
    '
```

## Backing Up Volume Metadata

Volume metadata (labels, driver options) should also be preserved:

```bash
podman volume inspect my-database-volume > /backups/my-database-volume-metadata.json
```

To recreate a volume with its original labels:

```bash
LABELS=$(cat /backups/my-database-volume-metadata.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
labels = data[0].get('Labels', {})
for k, v in labels.items():
    print(f'--label {k}={v}', end=' ')
")

podman volume create $LABELS my-database-volume
```

## Backup Verification

Always verify that backups contain the expected data:

```bash
#!/bin/bash

BACKUP_FILE="$1"

echo "Verifying backup: $BACKUP_FILE"

# Check archive integrity
if ! gzip -t "$BACKUP_FILE" 2>/dev/null; then
    echo "FAIL: Archive is corrupted"
    exit 1
fi

# List contents
FILE_COUNT=$(tar tzf "$BACKUP_FILE" | wc -l)
echo "Files in archive: $FILE_COUNT"

# Check for expected files (customize per volume)
tar tzf "$BACKUP_FILE" | head -20
echo "..."

echo "Verification passed"
```

## Retention Policy

Implement a retention policy to avoid filling up your backup storage:

```bash
#!/bin/bash

BACKUP_ROOT="/backups/podman-volumes"
KEEP_DAYS=30
KEEP_MIN=5

# Remove old backups but keep at least KEEP_MIN
BACKUP_COUNT=$(ls -d "$BACKUP_ROOT"/*/ 2>/dev/null | wc -l)

if [ "$BACKUP_COUNT" -gt "$KEEP_MIN" ]; then
    find "$BACKUP_ROOT" -maxdepth 1 -type d -mtime +$KEEP_DAYS | \
        sort | head -n -$KEEP_MIN | xargs rm -rf
fi
```

## Conclusion

Volume backups are the most critical part of any container backup strategy because volumes hold the data that cannot be recreated. Use application-level dump tools for databases, tar archives for general volumes, and always verify that your backups can be restored. The few minutes spent setting up automated volume backups will save you from the hours or days of pain that follow data loss.
