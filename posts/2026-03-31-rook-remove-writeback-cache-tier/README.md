# How to Remove a Writeback Cache Tier in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Storage, Operations

Description: Learn how to safely remove a writeback cache tier from Ceph by flushing all dirty objects, transitioning through readproxy mode, and then removing the tier relationship.

---

Removing a writeback cache tier requires careful handling because dirty objects in the cache pool have not yet been written to the backing pool. Deleting the cache pool before flushing these objects would cause permanent data loss. The safe removal procedure transitions through readproxy mode and waits for all dirty objects to be flushed.

## Understanding the Risk

In writeback mode, dirty objects exist only in the cache pool. If you delete the cache pool while dirty objects remain:
- Those writes are permanently lost
- The backing pool does not have those objects
- Data loss is unrecoverable

Always complete the full flush procedure before deleting the cache pool.

## Step 1: Transition to Readproxy Mode

Stop accepting new writes to the cache pool by switching to readproxy:

```bash
ceph osd tier cache-mode cache-pool readproxy
```

After this change:
- New client writes go directly to the backing pool
- No new dirty objects are created in the cache pool
- Existing dirty objects continue to be flushed

## Step 2: Force Flush All Dirty Objects

Trigger immediate flushing of all remaining dirty objects:

```bash
rados -p cache-pool cache-flush-evict-all
```

This command reads all objects in the cache pool and flushes dirty ones to the backing pool.

Monitor progress:

```bash
watch "ceph df | grep cache-pool"
```

Wait until the dirty count drops to 0. This may take minutes to hours depending on cache size and I/O subsystem speed.

## Step 3: Verify No Dirty Objects Remain

```bash
# Check for dirty objects
rados -p cache-pool ls | wc -l
```

Also check via pool stats:

```bash
ceph osd pool stats cache-pool
```

```text
cache tier io 0 MiB/s flush, 0 MiB/s promote
```

Zero flush rate with a decreasing or zero object count confirms flushing is complete.

## Step 4: Evict All Remaining Clean Objects

Remove all clean objects from the cache pool:

```bash
rados -p cache-pool cache-evict-all
```

Wait for the pool to empty:

```bash
watch "ceph df | grep cache-pool"
```

The object count should reach 0.

## Step 5: Remove Overlay and Tier Relationship

```bash
# Remove the overlay
ceph osd tier remove-overlay backing-pool

# Remove the tier relationship
ceph osd tier remove backing-pool cache-pool
```

## Step 6: Delete the Cache Pool

```bash
ceph config set mon mon_allow_pool_delete true
ceph osd pool delete cache-pool cache-pool --yes-i-really-really-mean-it
```

## Complete Safe Removal Script

```bash
#!/bin/bash
set -e
BACKING=backing-pool
CACHE=cache-pool

echo "Step 1: Switching to readproxy mode..."
ceph osd tier cache-mode $CACHE readproxy

echo "Step 2: Flushing all dirty objects..."
rados -p $CACHE cache-flush-evict-all

echo "Step 3: Waiting for flush completion..."
while true; do
  DIRTY=$(ceph df detail --format json | python3 -c "
import json, sys
df = json.load(sys.stdin)
for pool in df['pools']:
    if pool['name'] == '$CACHE':
        print(pool.get('stats', {}).get('dirty', 0))
")
  echo "Dirty objects remaining: $DIRTY"
  [ "$DIRTY" = "0" ] && break
  sleep 30
done

echo "Step 4: Evicting clean objects..."
rados -p $CACHE cache-evict-all

echo "Step 5: Removing overlay and tier..."
ceph osd tier remove-overlay $BACKING
ceph osd tier remove $BACKING $CACHE

echo "Step 6: Deleting cache pool..."
ceph config set mon mon_allow_pool_delete true
ceph osd pool delete $CACHE $CACHE --yes-i-really-really-mean-it

echo "Removal complete. Cluster status:"
ceph health
```

## Summary

Removing a writeback cache tier requires transitioning to readproxy mode to stop new dirty writes, flushing all existing dirty objects to the backing pool, evicting all clean objects, then removing the overlay, tier relationship, and finally the pool. Never skip the flush step - dirty objects in the cache pool that are not in the backing pool will be permanently lost if the cache pool is deleted prematurely.
