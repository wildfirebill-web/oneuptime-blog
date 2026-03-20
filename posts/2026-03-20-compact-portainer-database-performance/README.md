# How to Compact the Portainer Database for Better Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Performance, Database, Maintenance, BoltDB

Description: Learn how to compact and optimize the Portainer BoltDB database to reclaim disk space and improve performance on busy instances.

---

Portainer uses BoltDB as its embedded database. Over time, especially on active instances with many containers, logs, and stacks, the database can grow and accumulate unused space. Compacting it reclaims disk space and can improve read/write performance.

## Understanding Portainer's Database

The Portainer database is stored at `/data/portainer.db` inside the data volume. BoltDB is an append-only database-deleted records leave free space in the file that isn't automatically reclaimed. The compaction process rewrites the database, removing this unused space.

## Check Current Database Size

```bash
# Check the size of the portainer.db file

docker run --rm \
  -v portainer_data:/data \
  alpine \
  ls -lh /data/portainer.db
```

## Method 1: Use Portainer's Built-in Compact Feature (BE)

Portainer Business Edition exposes a database compaction endpoint:

```bash
# Authenticate
TOKEN=$(curl -s -X POST \
  https://localhost:9443/api/auth \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"yourpassword"}' \
  --insecure | python3 -c "import sys,json; print(json.load(sys.stdin)['jwt'])")

# Trigger database compaction
curl -X POST \
  https://localhost:9443/api/system/db/compact \
  -H "Authorization: Bearer $TOKEN" \
  --insecure

echo "Database compaction triggered"
```

## Method 2: Manual BoltDB Compaction (CE and BE)

For Community Edition or when the API is unavailable, use the `bbolt` tool:

```bash
# Step 1: Stop Portainer to ensure no writes during compaction
docker stop portainer

# Step 2: Record size before compaction
echo "Before compaction:"
docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db

# Step 3: Run bbolt compact using a temporary container
docker run --rm \
  -v portainer_data:/data \
  --entrypoint "" \
  portainer/portainer-ce:latest \
  /bin/sh -c "cp /data/portainer.db /data/portainer.db.bak && bbolt compact -o /data/portainer_compacted.db /data/portainer.db && mv /data/portainer_compacted.db /data/portainer.db"

# Step 4: Check size after compaction
echo "After compaction:"
docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db

# Step 5: Restart Portainer
docker start portainer
```

## Method 3: Export and Reimport (Nuclear Option)

For severely fragmented databases, a clean export/import achieves maximum compaction:

```bash
# Stop Portainer
docker stop portainer

# Back up the database
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine cp /data/portainer.db /backup/portainer_precompact.db

# Restart Portainer to let it self-compact on restart
docker start portainer

# Check if size improved after clean restart
sleep 10
docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db
```

## Compaction Schedule Recommendation

For production Portainer instances, compact the database:
- Monthly on active instances with many environments
- Quarterly on small installations
- Always before and after major version upgrades

```bash
#!/bin/bash
# Add to monthly cron: 0 3 1 * * /usr/local/bin/portainer-compact.sh
docker stop portainer
docker run --rm -v portainer_data:/data alpine \
  sh -c "ls -lh /data/portainer.db"
docker start portainer
```

---

*Monitor Portainer's performance and disk usage with [OneUptime](https://oneuptime.com) infrastructure monitoring.*
