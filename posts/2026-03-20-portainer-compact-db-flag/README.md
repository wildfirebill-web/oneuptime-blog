# How to Use the --compact-db Flag to Compress Portainer Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, CLI, Database, Performance, Maintenance

Description: Use the --compact-db flag to compact Portainer's BoltDB database, reclaiming disk space and improving performance after heavy usage or large event accumulation.

## Introduction

Portainer uses BoltDB, an embedded key-value database. Like many databases, BoltDB doesn't automatically reclaim freed space when data is deleted. Over time, as stacks are removed, containers are deleted, and notifications accumulate and are cleared, the database file grows with unused (free) pages. The `--compact-db` flag rebuilds the database to reclaim this space.

## When to Use --compact-db

Use database compaction when:
- Portainer's database is unexpectedly large (check with `ls -lh /data/portainer.db`)
- Portainer UI feels sluggish and the database is on HDD
- After removing a large number of environments, stacks, or users
- As part of regular maintenance (monthly)
- After clearing many notifications or activity logs

## Step 1: Check Current Database Size

```bash
# Check the current size of portainer.db

docker run --rm \
  -v portainer_data:/data \
  alpine ls -lh /data/portainer.db

# Example output:
# -rw------- 1 root root 245M /data/portainer.db
# A 245MB database is candidate for compaction

# Check total volume size
docker volume inspect portainer_data --format '{{.Mountpoint}}' | xargs du -sh
```

## Step 2: Stop Portainer

Compaction requires Portainer to be stopped. BoltDB does not support online compaction:

```bash
# Stop and remove the container (preserving the data volume)
docker stop portainer
docker rm portainer

# Verify Portainer is stopped
docker ps | grep portainer  # Should return nothing
```

## Step 3: Run --compact-db

```bash
# Run the compact-db operation
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# This will:
# 1. Open the existing portainer.db
# 2. Create a new database with only live data (no free pages)
# 3. Replace the old database with the compacted version
# 4. Exit

# Note: Depending on database size, this can take 30 seconds to several minutes
```

## Step 4: Verify Space Reclaimed

```bash
# Check the new database size
docker run --rm \
  -v portainer_data:/data \
  alpine ls -lh /data/portainer.db

# Compare with the size from Step 1
# You should see a significant reduction
```

## Step 5: Restart Portainer

```bash
# Start Portainer normally
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

# Verify it started successfully
docker logs portainer --tail 20

# Test login
curl -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}'
```

## Step 6: Automate Monthly Compaction

```bash
#!/bin/bash
# compact-portainer-db.sh
# Schedule: 0 3 1 * * /opt/scripts/compact-portainer-db.sh

LOG_FILE="/var/log/portainer-compact.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Starting Portainer database compaction" >> $LOG_FILE

# Get DB size before
SIZE_BEFORE=$(docker run --rm \
  -v portainer_data:/data \
  alpine ls -l /data/portainer.db | awk '{print $5}')

echo "[$DATE] Database size before: $SIZE_BEFORE bytes" >> $LOG_FILE

# Stop Portainer
echo "[$DATE] Stopping Portainer..." >> $LOG_FILE
docker stop portainer
docker rm portainer

# Backup before compaction
docker run --rm \
  -v portainer_data:/data \
  -v /opt/backups:/backup \
  alpine cp /data/portainer.db "/backup/portainer-precompact-$(date +%Y%m%d).db"

# Run compaction
echo "[$DATE] Running compact-db..." >> $LOG_FILE
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Get DB size after
SIZE_AFTER=$(docker run --rm \
  -v portainer_data:/data \
  alpine ls -l /data/portainer.db | awk '{print $5}')

echo "[$DATE] Database size after: $SIZE_AFTER bytes" >> $LOG_FILE
SAVINGS=$((SIZE_BEFORE - SIZE_AFTER))
echo "[$DATE] Space reclaimed: $SAVINGS bytes" >> $LOG_FILE

# Start Portainer
echo "[$DATE] Starting Portainer..." >> $LOG_FILE
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

sleep 10

# Verify Portainer is running
if docker ps | grep -q portainer; then
  echo "[$DATE] Portainer started successfully" >> $LOG_FILE
else
  echo "[$DATE] ERROR: Portainer failed to start!" >> $LOG_FILE
fi
```

## Step 7: Use with Docker Compose

```yaml
# maintenance-compact.yml - run separately for maintenance
version: "3.8"
services:
  portainer-compact:
    image: portainer/portainer-ce:latest
    command: --compact-db
    volumes:
      - portainer_data:/data
    # This service exits when done

volumes:
  portainer_data:
    external: true  # Use existing volume
```

Run with:

```bash
# Stop Portainer first
docker compose -f docker-compose.yml stop portainer

# Run compaction
docker compose -f maintenance-compact.yml up
docker compose -f maintenance-compact.yml rm -f

# Start Portainer again
docker compose -f docker-compose.yml start portainer
```

## Important Caveats

```bash
# 1. ALWAYS backup before compaction
docker run --rm \
  -v portainer_data:/data \
  -v /tmp:/backup \
  alpine cp /data/portainer.db /tmp/portainer.db.backup

# 2. Compaction is NOT the same as clearing data
# It only reclaims free space - it doesn't delete any active data

# 3. Portainer MUST be stopped during compaction
# Running compact-db on a running database can corrupt it

# 4. The --compact-db flag uses the SAME data volume as regular Portainer
# It reads from portainer_data:/data and writes back to it
```

## Conclusion

The `--compact-db` flag is a simple maintenance operation that reclaims wasted space in Portainer's BoltDB database without affecting any active data. Run it monthly or whenever the database file grows unexpectedly large. Always stop Portainer, create a backup, run compaction, verify the results, then restart - the whole process takes under 5 minutes and can recover significant disk space in active Portainer installations.
