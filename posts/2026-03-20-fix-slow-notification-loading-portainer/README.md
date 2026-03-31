# How to Fix Slow Notification Loading Affecting Bulk Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Troubleshooting, Performance, Notification, Bulk Operation, Optimization

Description: Learn how to fix Portainer's slow notification loading that blocks bulk container and stack operations, including notification queue management and database optimization.

---

In environments with high activity, Portainer's notification system can accumulate a large queue. When the notification system is slow to process, bulk operations (like restarting multiple containers) appear to hang while waiting for notifications to acknowledge.

## Understanding the Problem

Portainer stores activity logs and notifications in BoltDB. A large notification history causes:

- Slow response on bulk restart/stop/start operations
- UI freezes after performing actions on many containers
- Long delays before the operation confirmation appears

## Step 1: Check Notification Queue Size

```bash
# Inspect the Portainer data volume for database size

docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db

# A database file over 500MB often indicates notification accumulation
```

## Step 2: Clear Old Notifications

In Portainer go to **Notifications** (bell icon) and clear old entries. For large queues, this may need to be done in batches.

## Step 3: Compact the Database

Use the `--compact-db` flag to compress the BoltDB database:

```bash
# Stop Portainer
docker stop portainer

# Run compaction (this rewrites the database removing free space)
docker run --rm -v portainer_data:/data \
  portainer/portainer-ce:latest \
  --compact-db

# Restart Portainer
docker start portainer

# Verify the database shrank
docker run --rm -v portainer_data:/data alpine ls -lh /data/portainer.db
```

## Step 4: Limit Activity Log Retention (Business Edition)

In Portainer Business Edition, configure activity log retention:

1. Go to **Settings > General**.
2. Under **Activity Logging**, set a retention period (e.g., 30 days).
3. Click **Save settings**.

This prevents unbounded log growth.

## Step 5: Reduce Concurrent Bulk Operations

Instead of selecting all containers and restarting simultaneously, process them in smaller batches:

```bash
# Use Docker CLI for bulk operations on many containers
# More reliable than Portainer UI for 50+ containers
docker ps -q | xargs -P 4 -I {} docker restart {}
# -P 4 means 4 parallel restarts
```

## Step 6: Monitor Database Growth

Add a monitoring check to alert when the Portainer database grows unexpectedly:

```bash
# Add to a cron job to alert when db exceeds 200MB
DB_SIZE=$(docker run --rm -v portainer_data:/data alpine stat -c%s /data/portainer.db)
if [ $DB_SIZE -gt 209715200 ]; then
  echo "Portainer DB exceeds 200MB: consider compaction"
fi
```
