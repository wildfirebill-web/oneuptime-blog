# How to Identify and Clean Up Unused Volumes in Portainer - Cleanup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Volumes, Cleanup, DevOps

Description: Learn how to identify unused Docker volumes in Portainer and safely clean them up to recover disk space.

## Introduction

Unused volumes accumulate when containers are removed without their volumes, when stacks are deleted without cleaning up storage, or during development iterations. These orphaned volumes can consume significant disk space. Portainer and Docker CLI provide tools to identify and remove them safely.

## Prerequisites

- Portainer installed with a connected Docker environment

## Understanding Unused Volumes

A volume is "unused" when no containers (running or stopped) reference it. This happens when:

1. Container removed but volume wasn't (default behavior - volumes persist).
2. Stack removed without the `--volumes` flag.
3. Named volume was created but never assigned to a container.
4. Container recreated with a different volume name.

## Step 1: Identify Unused Volumes via Portainer

1. Navigate to **Volumes** in Portainer.
2. Look at the **Containers** column - volumes with 0 containers are unused.
3. Or filter by unused volumes if the UI supports it.

## Step 2: Identify Unused Volumes via CLI

```bash
# List volumes with their usage:

docker volume ls

# Filter to show only unused volumes:
# (volumes not referenced by any container, running or stopped)
docker volume ls -f dangling=true

# More detailed view with container references:
docker volume ls --format "{{.Name}}" | while read volume; do
    # Count containers using this volume
    container_count=$(docker ps -aq --filter "volume=${volume}" | wc -l)
    all_container_count=$(docker ps -aq -f volume="${volume}" | wc -l)

    if [ "$container_count" -eq 0 ] && [ "$all_container_count" -eq 0 ]; then
        size=$(docker run --rm -v "${volume}:/vol" alpine du -sh /vol 2>/dev/null | cut -f1)
        echo "UNUSED: ${volume} (${size})"
    else
        echo "IN USE: ${volume} (${container_count} containers)"
    fi
done
```

## Step 3: Check Volume Contents Before Removal

Before removing, check if there's important data:

```bash
#!/bin/bash
# audit-unused-volumes.sh
# Shows details of unused volumes before cleanup

echo "=== Unused Volume Audit ==="
echo ""

# Get all unused volumes
UNUSED_VOLUMES=$(docker volume ls -f dangling=true -q)

if [ -z "${UNUSED_VOLUMES}" ]; then
    echo "No unused volumes found."
    exit 0
fi

TOTAL_SIZE=0

for volume in ${UNUSED_VOLUMES}; do
    # Get volume size
    SIZE=$(docker run --rm -v "${volume}:/vol" alpine du -sh /vol 2>/dev/null | cut -f1)

    # Get volume creation date
    CREATED=$(docker volume inspect "${volume}" --format '{{.CreatedAt}}')

    # Get volume labels
    LABELS=$(docker volume inspect "${volume}" --format '{{json .Labels}}')

    # List top-level files
    FILES=$(docker run --rm -v "${volume}:/vol" alpine ls /vol 2>/dev/null | head -5 | tr '\n' ' ')

    echo "Volume: ${volume}"
    echo "  Created: ${CREATED}"
    echo "  Size: ${SIZE}"
    echo "  Labels: ${LABELS}"
    echo "  Contents: ${FILES}"
    echo ""
done

echo "Run 'docker volume prune' to remove all unused volumes."
echo "Run 'docker volume rm <name>' to remove specific volumes."
```

## Step 4: Remove Unused Volumes via Portainer

### Remove Individual Unused Volumes

1. Navigate to **Volumes** in Portainer.
2. Click **Remove** (trash icon) on an unused volume.
3. Confirm deletion.

### Bulk Remove via Prune

1. Navigate to **Volumes** in Portainer.
2. Click **Prune** or **Clean up unused volumes**.
3. Confirm.

## Step 5: Remove via Docker CLI

```bash
# Remove all unused volumes (no container references):
docker volume prune

# With confirmation bypass:
docker volume prune --force

# Remove volumes by label (e.g., test volumes):
docker volume prune --filter "label=environment=test" --force

# Remove volumes older than a certain date:
docker volume prune --filter "until=720h" --force  # 30 days

# Remove specific volumes:
docker volume rm old_app_data old_cache_data
```

## Step 6: Find and Remove Development Volumes

Development environments generate many temporary volumes:

```bash
#!/bin/bash
# cleanup-dev-volumes.sh
# Remove volumes from development/test environments

# Pattern 1: Remove volumes with test/dev labels
docker volume ls --filter "label=environment=test" -q | \
    xargs -r docker volume rm

docker volume ls --filter "label=environment=development" -q | \
    xargs -r docker volume rm

# Pattern 2: Remove volumes matching naming patterns
docker volume ls -q | grep -E "_test_|_dev_|_tmp_" | \
    while read vol; do
        # Verify it's not in use:
        IN_USE=$(docker ps -aq --filter "volume=${vol}" | wc -l)
        if [ "$IN_USE" -eq 0 ]; then
            echo "Removing: ${vol}"
            docker volume rm "${vol}"
        fi
    done
```

## Step 7: Automated Cleanup with Safety Checks

```bash
#!/bin/bash
# safe-volume-cleanup.sh
# Removes unused volumes with size and age checks

SAFE_MIN_AGE_DAYS="${1:-7}"    # Don't remove volumes newer than 7 days
LOG_FILE="/var/log/volume-cleanup.log"

echo "=== Volume Cleanup: $(date) ===" >> "${LOG_FILE}"

# Get dangling volumes
for volume in $(docker volume ls -f dangling=true -q); do
    # Get creation date (Docker volume timestamps in RFC3339 format)
    CREATED_AT=$(docker volume inspect "${volume}" --format '{{.CreatedAt}}')

    # Calculate age in days (simplified - uses date command)
    CREATED_EPOCH=$(date -d "${CREATED_AT}" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%S" "${CREATED_AT%.*}" "+%s" 2>/dev/null)
    NOW_EPOCH=$(date +%s)
    AGE_DAYS=$(( (NOW_EPOCH - CREATED_EPOCH) / 86400 ))

    if [ "${AGE_DAYS}" -lt "${SAFE_MIN_AGE_DAYS}" ]; then
        echo "SKIP (too new, ${AGE_DAYS}d): ${volume}" >> "${LOG_FILE}"
        continue
    fi

    # Get size
    SIZE=$(docker run --rm -v "${volume}:/vol" alpine du -sh /vol 2>/dev/null | cut -f1)

    echo "REMOVING (${AGE_DAYS}d old, ${SIZE}): ${volume}" >> "${LOG_FILE}"
    docker volume rm "${volume}" >> "${LOG_FILE}" 2>&1
done

echo "Cleanup complete." >> "${LOG_FILE}"
echo "" >> "${LOG_FILE}"
```

## Step 8: Monitor Volume Disk Usage

Prevent disk exhaustion by monitoring volume usage:

```bash
#!/bin/bash
# monitor-volume-usage.sh
# Alerts if total volume usage exceeds threshold

MAX_GB_THRESHOLD=50
ALERT_WEBHOOK="${SLACK_WEBHOOK:-}"

# Total Docker disk usage
VOLUME_USAGE=$(docker system df --format '{{.Size}}' | tail -1)
echo "Total volume usage: ${VOLUME_USAGE}"

# Per-volume breakdown:
echo ""
echo "Top 10 volumes by size:"
docker volume ls -q | while read vol; do
    size=$(docker run --rm -v "${vol}:/vol" alpine du -sk /vol 2>/dev/null | awk '{print $1}')
    echo "${size} ${vol}"
done | sort -rn | head -10 | while read size vol; do
    echo "  $(( size / 1024 ))MB: ${vol}"
done
```

## Conclusion

Keeping Docker volumes clean requires regular auditing and pruning. Use `docker volume prune` for quick bulk cleanup of volumes with no container references, but check volume contents before mass deletion. For safe automated cleanup, implement age and size checks, and label volumes by environment (test, dev, production) to enable targeted cleanup. In production, implement a process to review unused volumes before removal to avoid data loss.
