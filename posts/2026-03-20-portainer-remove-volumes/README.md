# How to Remove Volumes in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Cleanup, DevOps

Description: Learn how to safely remove Docker volumes in Portainer, including how to avoid accidental data loss and clean up unused volumes.

## Introduction

Docker volumes persist data independently of containers — this is their primary advantage. But when volumes are no longer needed, they take up disk space and clutter the Portainer UI. Removing volumes correctly requires care to avoid data loss. This guide covers safe volume removal practices.

## Prerequisites

- Portainer installed with a connected Docker environment

## Important Warning

Removing a Docker volume permanently deletes all data stored in it. Unlike containers, there's no "restore from stopped" option. Always:

1. Verify the volume is not needed before deleting.
2. Back up the volume data if there's any doubt.
3. Ensure no containers are using the volume.

## Step 1: Remove a Volume via Portainer

1. Navigate to **Volumes** in Portainer.
2. Find the volume to remove.
3. Click the **Remove** button (trash icon).
4. Confirm the removal.

Portainer will refuse to remove volumes that are currently in use by containers (running or stopped).

## Step 2: Remove Multiple Volumes

1. Navigate to **Volumes**.
2. Check the checkboxes next to volumes to remove.
3. Click **Remove** in the bulk action bar.

## Step 3: Remove Unused Volumes (Prune)

Remove all volumes not attached to any container:

1. Navigate to **Volumes** in Portainer.
2. Click **Prune** (if available).

Via Docker CLI:

```bash
# Remove all unused volumes (not attached to any container):
docker volume prune

# WARNING: "unused" includes volumes from STOPPED containers
# If you removed a container but might start it again, this removes its volume

# Without confirmation:
docker volume prune --force

# Remove volumes with specific labels:
docker volume prune --filter "label=environment=test"
```

## Step 4: Verify Volume Is Safe to Remove

Before removing, check what's using a volume:

```bash
# Check which containers use a volume:
docker ps -aq | xargs docker inspect --format \
  '{{.Name}} {{range .Mounts}}{{.Name}}{{end}}' | \
  grep "my-volume-name"

# Or specifically:
docker ps -aq --filter "volume=my-volume-name"
# If empty: no running containers use this volume

# Check stopped containers too:
docker ps -aq --filter "volume=my-volume-name" --filter "status=exited"

# Check volume size before deleting:
docker run --rm \
  -v my-volume-name:/data \
  alpine:latest \
  du -sh /data
```

## Step 5: Back Up Before Removing

Always back up important data before deleting:

```bash
#!/bin/bash
# backup-volume.sh
# Back up a volume before removing it

VOLUME_NAME="${1:?Volume name required}"
BACKUP_DIR="${2:-/backups/volumes}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${VOLUME_NAME}_${TIMESTAMP}.tar.gz"

mkdir -p "${BACKUP_DIR}"

echo "Backing up volume: ${VOLUME_NAME}"
echo "Destination: ${BACKUP_FILE}"

docker run --rm \
  -v "${VOLUME_NAME}:/data:ro" \
  -v "${BACKUP_DIR}:/backup" \
  alpine:latest \
  tar czf "/backup/${VOLUME_NAME}_${TIMESTAMP}.tar.gz" -C /data .

echo "Backup size: $(du -sh ${BACKUP_FILE} | cut -f1)"
echo "Backup complete: ${BACKUP_FILE}"
echo ""
echo "To restore: docker run --rm -v ${VOLUME_NAME}:/data -v ${BACKUP_DIR}:/backup alpine:latest tar xzf /backup/${VOLUME_NAME}_${TIMESTAMP}.tar.gz -C /data"
```

## Step 6: Remove Volume and Associated Container Together

When removing a temporary service (container + volume):

```bash
# Remove container and its anonymous volumes:
docker rm --volumes my-container

# For named volumes, you must remove separately:
docker rm my-container
docker volume rm my-named-volume

# Remove via docker-compose (removes containers AND named volumes):
docker compose down --volumes
# WARNING: Permanently deletes all named volumes defined in the compose file
```

In Portainer when removing a stack:
1. Navigate to **Stacks**.
2. Click the stack name.
3. Click **Remove this stack**.
4. Check **Remove associated volumes** (if data deletion is intended).

## Step 7: Volume Removal for Different Scenarios

### After a Failed Deployment

```bash
# Deployment created volumes but failed — clean up:
docker compose down --volumes
# Removes all containers, networks, and volumes defined in the compose file
```

### Development Environment Cleanup

```bash
# Remove all test volumes (labeled as test environment):
docker volume ls --filter "label=environment=test" -q | \
    xargs docker volume rm 2>/dev/null || true

# Nuclear option: remove everything in dev:
docker system prune --volumes --force
# WARNING: Removes ALL unused containers, networks, images, and volumes
```

### Production Volume Cleanup

```bash
# Only remove a specific volume after verifying it's empty and unused:
VOLUME="old-app-data-v1"

# 1. Verify no containers use it:
USERS=$(docker ps -aq --filter "volume=${VOLUME}" | wc -l)
[ "$USERS" -gt 0 ] && echo "ERROR: Volume is in use!" && exit 1

# 2. Check it's actually empty:
SIZE=$(docker run --rm -v "${VOLUME}:/data" alpine du -sk /data | cut -f1)
[ "$SIZE" -gt 100 ] && echo "WARNING: Volume has ${SIZE} KB of data!" && read -p "Delete anyway? [y/N] " confirm
[ "$confirm" != "y" ] && [ "$confirm" != "Y" ] && exit 0

# 3. Remove
docker volume rm "${VOLUME}"
echo "Volume removed: ${VOLUME}"
```

## Step 8: Automating Volume Cleanup

```bash
#!/bin/bash
# volume-cleanup.sh
# Remove volumes with cleanup labels and past their retention date

# Remove volumes labeled for cleanup:
docker volume ls --filter "label=auto-cleanup=true" -q | \
    while read volume; do
        # Check if created more than 30 days ago:
        CREATED=$(docker volume inspect "${volume}" --format '{{.CreatedAt}}')
        # (Parse date and compare — simplified here)
        echo "Removing cleanup-labeled volume: ${volume}"
        docker volume rm "${volume}" 2>/dev/null || echo "  ${volume} is in use — skipped"
    done
```

## Common Errors When Removing Volumes

```bash
# Error: "volume is in use"
# Fix: find and remove/stop the container using it
docker ps -aq --filter "volume=my-volume"
docker stop <container_id>
docker rm <container_id>
docker volume rm my-volume

# Error: "volume not found"
# Fix: verify the exact volume name
docker volume ls | grep my-volume

# Error: "device or resource busy"
# Fix: unmount any external mounts
# Check: cat /proc/mounts | grep my-volume
```

## Conclusion

Removing volumes in Portainer is permanent and irreversible — always back up data before deletion. Use the prune command for batch cleanup of unused volumes, verify no containers are using a volume before removal, and label your volumes with retention metadata to make cleanup decisions easier. For production environments, implement a backup-before-remove policy using automation scripts.
