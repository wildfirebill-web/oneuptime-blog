# How to Clean Up Stale Data in the Portainer Database - Stale

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Database Cleanup, BoltDB, Performance, Maintenance, Administration

Description: Learn how to clean up stale data in Portainer's BoltDB database by removing old snapshots, compacting the database, and pruning unused resources.

---

Over time, Portainer's BoltDB database accumulates stale snapshot data, orphaned entries from deleted containers, and historical records that slow down the UI. Regular cleanup maintains performance.

## Understanding What Grows in the Database

The main sources of database growth:

| Data Type | Growth Rate | Cleanup Method |
|-----------|-------------|----------------|
| Docker snapshots | Each snapshot cycle | Automatic (replaced) |
| DockerSnapshotRaw | Per environment per cycle | `--compact-db` |
| Activity logs | Per user action | Time-based retention |
| Stack history | Per deployment | Manual pruning |
| Notification history | Per event | Manual pruning |

## Step 1: Check Database Size

```bash
# Size of the Portainer database file

docker exec portainer du -sh /data/portainer.db

# Or from outside the container
docker run --rm -v portainer_data:/data alpine du -sh /data/portainer.db
```

A healthy database for a medium deployment (10–50 containers) is typically 10–50 MB. If it's over 200 MB, cleanup is needed.

## Step 2: Compact the Database

BoltDB does not automatically release unused space. Compaction rewrites the database without fragmentation:

```bash
# Stop Portainer first
docker stop portainer

# Run compaction
docker run --rm -v portainer_data:/data \
  portainer/portainer-ce:latest --compact-db

# Restart
docker start portainer

# Verify size reduction
docker exec portainer du -sh /data/portainer.db
```

## Step 3: Prune Unused Docker Resources

Stale Docker resources (stopped containers, dangling images, unused volumes) increase snapshot size. Clean them up on each managed host:

```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune -f

# Also remove unused volumes (careful - ensure no data is needed)
docker volume prune -f

# Remove unused images (not just dangling)
docker image prune -a --filter "until=720h" -f   # Images unused for 30 days
```

## Step 4: Remove Unused Stacks from Portainer

Stacks that have been deleted from Docker but still appear in Portainer consume database entries:

```bash
# List all stacks via API
TOKEN=$(curl -s -X POST https://portainer.example.com/api/auth \
  -d '{"Username":"admin","Password":"pass"}' -H 'Content-Type: application/json' | jq -r .jwt)

curl -s -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/stacks | jq '.[] | {Id, Name, Status}'

# Delete a specific orphaned stack (ID from above)
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  https://portainer.example.com/api/stacks/7
```

## Step 5: Reduce Activity Log Retention

Portainer Business Edition allows configuring activity log retention:

1. Go to **Settings > Authentication** (BE only).
2. Set **Activity log retention** to 30 or 90 days instead of unlimited.

For Community Edition, the activity log doesn't persist between restarts and stays small.

## Step 6: Remove Disconnected Environments

Each disconnected environment still occupies snapshot storage. Remove environments you no longer use:

1. Go to **Environments**.
2. Click the trash icon next to unused/offline environments.
3. Or update the environment URL if the host IP changed.

## Automation: Weekly Cleanup Script

Schedule regular maintenance:

```bash
#!/bin/bash
# weekly-portainer-cleanup.sh

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# Prune Docker resources on the local host
log "Pruning Docker resources..."
docker system prune -f
docker volume prune -f

# Compact Portainer database
log "Compacting Portainer database..."
SIZE_BEFORE=$(docker exec portainer du -sh /data/portainer.db | cut -f1)
docker stop portainer
docker run --rm -v portainer_data:/data portainer/portainer-ce:latest --compact-db
docker start portainer
SIZE_AFTER=$(docker exec portainer du -sh /data/portainer.db | cut -f1)

log "Database compacted: $SIZE_BEFORE -> $SIZE_AFTER"
```

Add to cron: `0 3 * * 0 root /usr/local/bin/weekly-portainer-cleanup.sh`
