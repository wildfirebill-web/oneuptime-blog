# How to Remove a Read-Only Cache Tier in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CacheTiering, Storage, Operation

Description: Learn the steps to safely remove a read-only cache tier from a Ceph cluster, including removing the overlay, breaking the tier relationship, and deleting the cache pool.

---

Removing a read-only cache tier is straightforward because there are no dirty objects to flush - all writes went directly to the backing pool. The cache pool contains only clean copies of objects that can be safely discarded. The removal process involves three steps: removing the overlay, breaking the tier relationship, and deleting the cache pool.

## Before You Begin

Confirm the cache tier is in readonly mode:

```bash
ceph osd dump | grep cache_mode
```

Expected:

```text
cache_mode: readonly
```

If the cache is in writeback mode, follow the writeback removal procedure instead (which requires flushing dirty objects first).

## Step 1: Change Cache Mode to None

For safety, disable the cache mode before removing the overlay:

```bash
ceph osd tier cache-mode cache-pool none
```

This prevents any new promotions from occurring during the removal process.

## Step 2: Wait for In-Progress Operations

Allow any ongoing cache I/O to complete:

```bash
# Check for active PG operations
ceph pg stat
```

Wait until the cluster returns to `HEALTH_OK` with no active operations on the cache pool.

## Step 3: Remove the Overlay

The overlay is what directs client I/O through the cache pool. Removing it means clients will now access the backing pool directly:

```bash
ceph osd tier remove-overlay backing-pool
```

Verify the overlay is removed:

```bash
ceph osd dump | grep -A 3 "pool 'backing-pool'"
```

The output should no longer show `read_tier` or `write_tier` entries.

## Step 4: Remove the Tier Relationship

```bash
ceph osd tier remove backing-pool cache-pool
```

This breaks the parent-child relationship between the two pools. Verify:

```bash
ceph osd dump | grep tier
# Should show no tier relationships
```

## Step 5: Delete the Cache Pool

```bash
# First confirm the pool name
ceph osd pool ls

# Delete the cache pool (requires two confirmations)
ceph osd pool delete cache-pool cache-pool --yes-i-really-really-mean-it
```

If pool deletion is protected:

```bash
ceph config set mon mon_allow_pool_delete true
ceph osd pool delete cache-pool cache-pool --yes-i-really-really-mean-it
```

## Step 6: Verify Complete Removal

```bash
ceph osd pool ls
ceph osd dump | grep -i tier
ceph health
```

The cluster should return to `HEALTH_OK` with only the backing pool remaining.

## Complete Removal Script

```bash
#!/bin/bash
BACKING=backing-pool
CACHE=cache-pool

echo "Disabling cache mode..."
ceph osd tier cache-mode $CACHE none

echo "Waiting for cluster..."
sleep 10

echo "Removing overlay..."
ceph osd tier remove-overlay $BACKING

echo "Removing tier relationship..."
ceph osd tier remove $BACKING $CACHE

echo "Deleting cache pool..."
ceph config set mon mon_allow_pool_delete true
ceph osd pool delete $CACHE $CACHE --yes-i-really-really-mean-it

echo "Done. Cluster status:"
ceph health
```

## Post-Removal Considerations

After removing the cache tier, clients will access the backing pool directly at HDD performance. If the SSD capacity is being repurposed:
- The SSD OSDs can be reassigned to a different pool via CRUSH rules
- The CRUSH rule used by the cache pool can be deleted or repurposed

## Summary

Removing a read-only cache tier is safe and simple because there are no dirty objects. The process is: set cache mode to none, remove the overlay, break the tier relationship, then delete the cache pool. The entire operation can be completed in minutes with no data at risk since all data resides in the backing pool.
