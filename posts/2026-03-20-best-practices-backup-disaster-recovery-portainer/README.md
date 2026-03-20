# Best Practices for Backup and Disaster Recovery with Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Backup, Disaster Recovery, Docker, Best Practices, DevOps, Resilience

Description: Implement a comprehensive backup and disaster recovery strategy for Portainer itself and the containerized workloads it manages, including automated backup procedures and recovery runbooks.

---

A Portainer deployment without a backup strategy is a single point of failure. This guide covers backing up Portainer's configuration and the data volumes of your containerized applications, along with recovery procedures.

## What Needs to Be Backed Up

```bash
Portainer itself:
  - portainer_data volume (DB, configs, keys)
  - TLS certificates

Your workloads:
  - Named Docker volumes (database data, file uploads, etc.)
  - Stack definitions (best kept in Git)
  - Registry credentials
  - Environment variable configurations
```

## Backup Portainer's Data Volume

Portainer stores all configuration, user accounts, and environment data in the `portainer_data` volume. Back it up regularly:

```bash
#!/bin/bash
# backup-portainer.sh

BACKUP_DIR=/opt/backups/portainer
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

# Stop Portainer for a consistent backup (or use the API to export)

docker stop portainer

# Create a tarball of the volume
docker run --rm \
  -v portainer_data:/data:ro \
  -v "$BACKUP_DIR":/backups \
  alpine tar czf "/backups/portainer-data-$TIMESTAMP.tar.gz" /data

# Restart Portainer
docker start portainer

echo "Portainer backup completed: portainer-data-$TIMESTAMP.tar.gz"

# Keep only the last 30 backups
ls -t "$BACKUP_DIR"/portainer-data-*.tar.gz | tail -n +31 | xargs -r rm
```

## Backup Application Volumes

For each named volume used by your applications:

```bash
#!/bin/bash
# backup-volumes.sh

BACKUP_DIR=/opt/backups/volumes
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"

# List of critical volumes to back up
VOLUMES=(
  "postgres-data"
  "wordpress-uploads"
  "redis-data"
  "app-config"
)

for VOLUME in "${VOLUMES[@]}"; do
  echo "Backing up volume: $VOLUME"
  docker run --rm \
    -v "$VOLUME":/data:ro \
    -v "$BACKUP_DIR":/backups \
    alpine tar czf "/backups/$VOLUME-$TIMESTAMP.tar.gz" /data
done

echo "All volume backups complete"
```

## Use Portainer's Built-In Backup (BE)

Portainer Business Edition has a built-in backup feature:

1. Go to **Settings > Backup Portainer**
2. Enable scheduled backups
3. Set backup interval (daily recommended)
4. Configure S3 or local storage destination

This backs up the entire Portainer database including environments, stacks, users, and access policies.

## Offsite Backup with Restic

Use Restic to back up volumes to S3/B2/Wasabi:

```bash
#!/bin/bash
# restic-backup.sh

export RESTIC_REPOSITORY="s3:s3.amazonaws.com/my-backup-bucket/portainer"
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export RESTIC_PASSWORD="your-restic-password"

# Initialize repository (first run only)
# restic init

# Back up all critical volumes
docker run --rm \
  -v portainer_data:/backup/portainer:ro \
  -v postgres-data:/backup/postgres:ro \
  -e RESTIC_REPOSITORY \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  -e RESTIC_PASSWORD \
  restic/restic:latest \
  backup /backup --tag "daily"

# Prune old backups
docker run --rm restic/restic:latest \
  forget --keep-daily 7 --keep-weekly 4 --keep-monthly 3 --prune
```

## Recovery Runbook

Document and test your recovery procedure:

```bash
#!/bin/bash
# restore-portainer.sh

BACKUP_FILE="$1"
if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 /path/to/portainer-data-TIMESTAMP.tar.gz"
  exit 1
fi

# Stop existing Portainer
docker stop portainer

# Remove old volume (WARNING: this deletes all Portainer config)
docker volume rm portainer_data
docker volume create portainer_data

# Restore from backup
docker run --rm \
  -v portainer_data:/data \
  -v "$(dirname $BACKUP_FILE)":/backups:ro \
  alpine sh -c "cd / && tar xzf /backups/$(basename $BACKUP_FILE)"

# Restart Portainer
docker start portainer

echo "Portainer restored from $BACKUP_FILE"
echo "Verify at https://$(hostname):9443"
```

## Testing Backups

Test your backup recovery process quarterly:
1. Spin up a separate test server
2. Restore the Portainer backup
3. Verify all environments, stacks, and users are present
4. Verify application deployments work after volume restore

## RTO and RPO Planning

Define your recovery targets:

| Component | RPO (Max Data Loss) | RTO (Max Downtime) |
|-----------|--------------------|--------------------|
| Portainer config | 24 hours | 30 minutes |
| Database volumes | 1 hour | 1 hour |
| Application state | 15 minutes | 2 hours |

Adjust backup frequency to meet your RPO targets.

## Summary

Backup and disaster recovery for Portainer requires backing up both Portainer's own data volume and your application data volumes. Keep stack definitions in Git for zero-RPO configuration recovery. Test your recovery procedures regularly - a backup you've never restored is just hope.
