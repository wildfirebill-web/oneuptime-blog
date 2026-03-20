# How to Back Up Docker Volumes via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Backup, DevOps

Description: Learn how to back up Docker volumes in Portainer using temporary containers, automated scripts, and cloud storage integration.

## Introduction

Docker volumes contain your application's persistent data — databases, user uploads, configuration, and more. Unlike containers (which are disposable), volume data is critical and must be backed up. This guide covers practical approaches to backing up Docker volumes, from simple one-off backups to automated scheduled backups.

## Prerequisites

- Portainer installed with a connected Docker environment
- Named volumes to back up
- Storage destination (local directory, S3, NFS, etc.)

## The Core Backup Pattern

Since Docker volumes are directories on the host, the backup approach uses a temporary container to access and archive the volume:

```bash
# Core pattern: run a container, mount the volume + backup destination, create archive
docker run --rm \
  -v source_volume:/data:ro \
  -v /backup/destination:/backup \
  alpine:latest \
  tar czf /backup/backup-name.tar.gz -C /data .
```

## Method 1: One-Time Volume Backup

```bash
#!/bin/bash
# backup-volume.sh
# One-time backup of a named volume

VOLUME_NAME="${1:?Volume name required}"
BACKUP_DIR="${2:-/backups/volumes}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${VOLUME_NAME}_${TIMESTAMP}.tar.gz"

mkdir -p "${BACKUP_DIR}"

echo "Backing up volume: ${VOLUME_NAME}"
echo "Destination: ${BACKUP_DIR}/${BACKUP_FILE}"

# Create backup using Alpine container
docker run --rm \
  -v "${VOLUME_NAME}:/data:ro" \
  -v "${BACKUP_DIR}:/backup" \
  alpine:latest \
  tar czf "/backup/${BACKUP_FILE}" -C /data .

# Verify the backup was created:
if [ -f "${BACKUP_DIR}/${BACKUP_FILE}" ]; then
    SIZE=$(du -sh "${BACKUP_DIR}/${BACKUP_FILE}" | cut -f1)
    echo "✓ Backup created: ${BACKUP_FILE} (${SIZE})"
else
    echo "✗ Backup failed!"
    exit 1
fi
```

## Method 2: Consistent Backup (Pause Container First)

For databases and stateful apps, pause the container before backup:

```bash
#!/bin/bash
# consistent-backup.sh
# Pause container for consistent backup

CONTAINER_NAME="${1:?Container name required}"
VOLUME_NAME="${2:?Volume name required}"
BACKUP_DIR="${3:-/backups/volumes}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${VOLUME_NAME}_consistent_${TIMESTAMP}.tar.gz"

echo "Pausing container: ${CONTAINER_NAME}"
docker pause "${CONTAINER_NAME}"

echo "Creating consistent backup..."
docker run --rm \
  -v "${VOLUME_NAME}:/data:ro" \
  -v "${BACKUP_DIR}:/backup" \
  alpine:latest \
  tar czf "/backup/${BACKUP_FILE}" -C /data .

echo "Resuming container: ${CONTAINER_NAME}"
docker unpause "${CONTAINER_NAME}"

echo "✓ Backup complete: ${BACKUP_FILE}"
```

## Method 3: PostgreSQL Database Backup

For PostgreSQL, use `pg_dump` for a consistent backup:

```bash
#!/bin/bash
# backup-postgres.sh
# Creates a PostgreSQL logical backup

CONTAINER_NAME="${1:?PostgreSQL container required}"
DB_NAME="${2:?Database name required}"
DB_USER="${3:-postgres}"
BACKUP_DIR="${4:-/backups/postgres}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${DB_NAME}_${TIMESTAMP}.sql.gz"

mkdir -p "${BACKUP_DIR}"

echo "Backing up PostgreSQL database: ${DB_NAME}"

docker exec "${CONTAINER_NAME}" \
  pg_dump -U "${DB_USER}" -d "${DB_NAME}" --clean --if-exists | \
  gzip > "${BACKUP_DIR}/${BACKUP_FILE}"

SIZE=$(du -sh "${BACKUP_DIR}/${BACKUP_FILE}" | cut -f1)
echo "✓ Database backup complete: ${BACKUP_FILE} (${SIZE})"

# To restore:
echo ""
echo "To restore: zcat ${BACKUP_FILE} | docker exec -i ${CONTAINER_NAME} psql -U ${DB_USER} -d ${DB_NAME}"
```

## Method 4: Scheduled Automated Backups

Deploy a backup container as part of your stack:

```yaml
# backup-stack.yml
version: "3.8"

services:
  # Main application
  app:
    image: myorg/myapp:latest
    restart: unless-stopped
    volumes:
      - app_data:/app/data

  # PostgreSQL database
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=${DB_PASSWORD}

  # Automated backup service
  backup:
    image: ghcr.io/jamesbrink/backup:latest  # Or create your own
    restart: unless-stopped
    volumes:
      # Mount volumes to back up
      - postgres_data:/data/postgres:ro
      - app_data:/data/app:ro
      # Mount backup destination
      - /backups:/backups
    environment:
      # Run backup daily at 2 AM
      - BACKUP_SCHEDULE=0 2 * * *
      # Upload to S3
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_KEY}
      - S3_BUCKET=my-backups-bucket
      - S3_PREFIX=docker-volumes
      # Retention: keep 30 daily backups
      - BACKUP_RETENTION_DAYS=30

volumes:
  app_data:
  postgres_data:
```

## Method 5: Backup to S3

```bash
#!/bin/bash
# backup-to-s3.sh
# Backs up a Docker volume to AWS S3

VOLUME_NAME="${1:?Volume name required}"
S3_BUCKET="${S3_BUCKET:?S3_BUCKET env var required}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_KEY="docker-volumes/${VOLUME_NAME}/${VOLUME_NAME}_${TIMESTAMP}.tar.gz"

echo "Backing up ${VOLUME_NAME} to s3://${S3_BUCKET}/${BACKUP_KEY}"

# Pipe backup directly to S3 (no local temp file needed)
docker run --rm \
  -e AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
  -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
  -e AWS_DEFAULT_REGION="${AWS_DEFAULT_REGION:-us-east-1}" \
  -v "${VOLUME_NAME}:/data:ro" \
  amazon/aws-cli:latest \
  s3 cp - "s3://${S3_BUCKET}/${BACKUP_KEY}" \
  < <(docker run --rm -v "${VOLUME_NAME}:/data:ro" alpine tar czf - -C /data .)

echo "✓ Backup complete: s3://${S3_BUCKET}/${BACKUP_KEY}"
```

## Method 6: Portainer Edge Jobs for Automated Backups

Use Portainer Edge Jobs for scheduled backups on remote devices:

```bash
#!/bin/sh
# edge-backup-job.sh
# Runs as a Portainer Edge Job on edge devices

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DEVICE_ID="${DEVICE_ID:-unknown}"
BACKUP_DIR="/backup/${DEVICE_ID}"

mkdir -p "${BACKUP_DIR}"

# Backup all named volumes
for volume in app_data config_data; do
    docker run --rm \
      -v "${volume}:/data:ro" \
      -v "${BACKUP_DIR}:/backup" \
      alpine:latest \
      tar czf "/backup/${volume}_${TIMESTAMP}.tar.gz" -C /data .

    echo "Backed up: ${volume}"
done

# Cleanup backups older than 7 days
find "${BACKUP_DIR}" -name "*.tar.gz" -mtime +7 -delete
echo "Old backups cleaned up"

echo "Backup complete for device: ${DEVICE_ID}"
```

## Method 7: Restore from Backup

```bash
#!/bin/bash
# restore-volume.sh
# Restore a volume from a backup file

VOLUME_NAME="${1:?Volume name required}"
BACKUP_FILE="${2:?Backup file required}"

echo "Restoring volume: ${VOLUME_NAME} from ${BACKUP_FILE}"

# Create volume if it doesn't exist
docker volume create "${VOLUME_NAME}" 2>/dev/null || true

# Restore from backup
docker run --rm \
  -v "${VOLUME_NAME}:/data" \
  -v "$(dirname ${BACKUP_FILE}):/backup" \
  alpine:latest \
  tar xzf "/backup/$(basename ${BACKUP_FILE})" -C /data

echo "✓ Volume restored: ${VOLUME_NAME}"
echo "Start your containers to use the restored data"
```

## Conclusion

Docker volume backups are essential for data protection. The core pattern — running a temporary Alpine container to archive volume data — is simple and reliable. For production environments, implement scheduled backups using Portainer Edge Jobs or a dedicated backup container, upload to S3 for off-host storage, and regularly test your restore process. For databases, use native dump tools (pg_dump, mysqldump) rather than filesystem backups for consistency.
