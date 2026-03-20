# How to Clean Up Stale Data in the Portainer Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Maintenance, Database, Cleanup, Performance

Description: Remove stale containers, images, volumes, and endpoint data from Portainer's embedded database to improve performance and reclaim disk space.

## Introduction

Over time, Portainer's embedded boltdb database accumulates stale data: deleted endpoints that still have cached snapshots, old container records, removed images that are still referenced, and unused network configurations. This stale data increases database file size, slows queries, and consumes memory. This guide covers systematic cleanup at both the Docker layer and Portainer database layer.

## Step 1: Clean Up Docker Resources (Foundation)

Before cleaning Portainer's database, clean up the actual Docker resources:

```bash
# See what's consuming space

docker system df
# Shows:
# TYPE                TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images              45        12        15.6GB    12.1GB (77%)
# Containers          23        8         2.3GB     1.8GB (78%)
# Local Volumes       31        14        45.2GB    12.3GB (27%)
# Build Cache         156       0         8.2GB     8.2GB (100%)

# Remove stopped containers (Portainer will clean their references too)
docker container prune -f

# Remove unused images (not referenced by any container)
docker image prune -a -f

# Remove unused volumes (not mounted by any container)
docker volume prune -f

# Remove unused networks
docker network prune -f

# Remove build cache (safe to remove - rebuilds from scratch next time)
docker builder prune -a -f

# Complete cleanup (all of the above at once)
docker system prune -a -f --volumes

# Check reclaimed space
docker system df
```

## Step 2: Remove Stale Endpoints from Portainer

```bash
PORTAINER_URL="https://portainer.example.com"
TOKEN="your_api_token"

# List all endpoints
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/endpoints" | \
  jq '.[] | {id: .Id, name: .Name, status: .Status, type: .Type}'

# Status codes:
# 1 = Active
# 2 = Disconnected (stale)
# Remove disconnected endpoints

# Find disconnected endpoints
DISCONNECTED=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/endpoints" | \
  jq -r '.[] | select(.Status == 2) | .Id')

# Remove each disconnected endpoint
for endpoint_id in $DISCONNECTED; do
  echo "Removing stale endpoint: $endpoint_id"
  curl -s -X DELETE \
    -H "Authorization: Bearer $TOKEN" \
    "$PORTAINER_URL/api/endpoints/$endpoint_id"
done
```

## Step 3: Clean Up Old Stack Records

```bash
# List stacks in inactive or error state
curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/stacks" | \
  jq '.[] | {id: .Id, name: .Name, status: .Status}'
# Status: 1=Active, 2=Inactive

# Remove inactive stacks (compose deleted but record remains)
INACTIVE_STACKS=$(curl -s \
  -H "Authorization: Bearer $TOKEN" \
  "$PORTAINER_URL/api/stacks" | \
  jq -r '.[] | select(.Status == 2) | .Id')

for stack_id in $INACTIVE_STACKS; do
  echo "Removing inactive stack: $stack_id"
  curl -s -X DELETE \
    -H "Authorization: Bearer $TOKEN" \
    "$PORTAINER_URL/api/stacks/$stack_id"
done
```

## Step 4: Compact the boltdb Database

After removing data, the database file doesn't shrink automatically - it needs compaction:

```bash
#!/bin/bash
# compact-portainer-db.sh

echo "=== Portainer Database Compaction ==="

# Get current database size
DB_PATH=$(docker inspect portainer \
  --format '{{range .Mounts}}{{if eq .Destination "/data"}}{{.Source}}{{end}}{{end}}')

BEFORE_SIZE=$(du -sh "$DB_PATH/portainer.db" | cut -f1)
echo "Before: $BEFORE_SIZE"

# Stop Portainer (required for safe compaction)
echo "Stopping Portainer..."
docker stop portainer

# Compact using Docker's own utility or bbolt
# Method 1: Use alpine with bbolt (if available)
docker run --rm \
  -v "$DB_PATH:/data" \
  alpine:latest \
  sh -c "
    # Install bbolt tool
    wget -qO /tmp/bbolt https://github.com/etcd-io/bbolt/releases/download/v1.3.8/bbolt_1.3.8_linux_amd64 && \
    chmod +x /tmp/bbolt && \
    /tmp/bbolt compact -o /data/portainer.db.compact /data/portainer.db && \
    mv /data/portainer.db /data/portainer.db.bak && \
    mv /data/portainer.db.compact /data/portainer.db
  "

AFTER_SIZE=$(du -sh "$DB_PATH/portainer.db" | cut -f1)
echo "After: $AFTER_SIZE"

# Restart Portainer
echo "Starting Portainer..."
docker start portainer

echo "Compaction complete. Backup at: portainer.db.bak"
```

## Step 5: Automate Weekly Cleanup

```bash
# /etc/cron.weekly/portainer-cleanup

#!/bin/bash
LOG="/var/log/portainer-cleanup.log"
exec >> "$LOG" 2>&1

echo "=== Weekly Portainer Cleanup: $(date) ==="

# Step 1: Docker resource cleanup
echo "Cleaning Docker resources..."
docker container prune -f
docker image prune -f
docker volume prune -f
docker network prune -f

FREED=$(docker system df 2>&1 | grep "Build Cache" | awk '{print $NF}')
echo "Docker cleanup complete. Freed approximately $FREED"

# Step 2: Clean up old logs
find /var/lib/docker/containers -name "*.log" -size +100M | while read f; do
  echo "Truncating large log: $f"
  truncate -s 50M "$f"
done

# Step 3: Report
echo "Cleanup complete: $(date)"
docker system df

chmod +x /etc/cron.weekly/portainer-cleanup
```

## Step 6: Monitor Database Growth

```bash
#!/bin/bash
# monitor-db-growth.sh - Track database size over time

LOG_FILE="/var/log/portainer-db-size.log"
DB_PATH=$(docker inspect portainer \
  --format '{{range .Mounts}}{{if eq .Destination "/data"}}{{.Source}}{{end}}{{end}}')

while true; do
  SIZE=$(stat -c%s "$DB_PATH/portainer.db" 2>/dev/null || echo 0)
  CONTAINERS=$(docker ps -q | wc -l)
  echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) db_size_bytes=$SIZE running_containers=$CONTAINERS" \
    >> "$LOG_FILE"
  sleep 3600  # Log every hour
done

# Alert if database exceeds 500MB
DB_SIZE_MB=$(stat -c%s "$DB_PATH/portainer.db" | awk '{print int($1/1048576)}')
if [ "$DB_SIZE_MB" -gt 500 ]; then
  echo "WARNING: Portainer database is ${DB_SIZE_MB}MB - consider compaction"
fi
```

## Conclusion

Portainer database maintenance is a routine task, not a one-time fix. Docker resource cleanup (container prune, image prune, volume prune) is the first step - it removes the actual resources so Portainer's next snapshot reflects a clean state. Removing stale endpoints and inactive stack records via the Portainer API eliminates stale snapshot data. Weekly automated cleanup jobs prevent gradual accumulation. Schedule database compaction monthly or whenever the database grows beyond 200MB to keep Portainer responsive.
