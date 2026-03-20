# How to Compact the Portainer Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Database, Maintenance, BoltDB, Performance

Description: A guide to compacting the Portainer BoltDB database to reclaim disk space and improve performance.

## Overview

Portainer uses BoltDB as its embedded database to store configuration, activity logs, and metadata. Over time, as records are added and deleted, the database file can grow with unused space (fragmentation). Compacting the database reclaims this space. Portainer 2.17+ includes a built-in database compaction feature accessible via the command line.

## Prerequisites

- Portainer CE or Business Edition 2.17+
- Docker CLI access
- Admin access to the Docker host

## Why Compact the Database?

BoltDB uses a B-tree structure that does not automatically reclaim pages after deletions. Common causes of database bloat:

- Deleting many containers or stacks
- High activity logs accumulation
- Portainer upgrades that migrate data
- Long-running Portainer instances

```bash
# Check current database size

docker run --rm \
  -v portainer_data:/data \
  alpine \
  du -sh /data/portainer.db
```

## Step 1: Back Up Before Compacting

```bash
# Always back up before maintenance
docker stop portainer

docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine \
  cp /data/portainer.db /backup/portainer.db.bak-$(date +%Y%m%d)

ls -lh portainer.db.bak-*
```

## Step 2: Run Database Compaction

```bash
# Portainer 2.17+ supports --db-compact flag
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --db-compact

# For Business Edition
docker run --rm \
  -v portainer_data:/data \
  portainer/portainer-ee:latest \
  --db-compact
```

Expected output:
```text
2026/03/20 10:00:00 Starting Portainer 2.19.0
2026/03/20 10:00:01 Compacting database...
2026/03/20 10:00:03 Database compacted successfully. Old size: 45MB, New size: 12MB
```

## Step 3: Verify Compaction Results

```bash
# Compare database sizes
docker run --rm \
  -v portainer_data:/data \
  alpine \
  sh -c "du -sh /data/portainer.db && ls -lh /data/"
```

## Step 4: Restart Portainer

```bash
docker start portainer

# Verify Portainer is running correctly
docker logs portainer --tail 20
curl -k https://localhost:9443/api/status
```

## Alternative: Manual Compaction with bbolt

For older Portainer versions without `--db-compact`:

```bash
# Install bbolt CLI tool
go install go.etcd.io/bbolt/cmd/bbolt@latest

# Compact the database manually (requires Portainer stopped)
docker stop portainer

docker run --rm \
  -v portainer_data:/data \
  -v $(which bbolt || echo "/usr/local/go/bin/bbolt"):/usr/local/bin/bbolt \
  alpine \
  bbolt compact -o /data/portainer-compact.db /data/portainer.db

# Replace original with compacted
docker run --rm \
  -v portainer_data:/data \
  alpine \
  sh -c "mv /data/portainer.db /data/portainer.db.old && mv /data/portainer-compact.db /data/portainer.db"

docker start portainer
```

## Scheduling Regular Compaction

```bash
# Add to crontab for monthly compaction
# crontab -e
0 3 1 * * docker stop portainer && \
  docker run --rm -v portainer_data:/data portainer/portainer-ce:latest --db-compact && \
  docker start portainer
```

## Monitoring Database Size

```bash
# Script to alert if database exceeds threshold
#!/bin/bash
DB_SIZE=$(docker run --rm -v portainer_data:/data alpine du -sm /data/portainer.db | cut -f1)
THRESHOLD=100  # MB

if [ "${DB_SIZE}" -gt "${THRESHOLD}" ]; then
  echo "WARNING: Portainer database is ${DB_SIZE}MB (threshold: ${THRESHOLD}MB)"
  # Send alert notification here
fi
```

## Conclusion

Regular database compaction keeps Portainer running efficiently and prevents unnecessary disk consumption. Portainer 2.17+ makes this straightforward with the `--db-compact` flag. Run compaction during maintenance windows, always back up first, and consider scheduling monthly compaction for long-running Portainer instances with high activity.
