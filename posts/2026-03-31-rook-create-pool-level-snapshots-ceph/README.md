# How to Create Pool-Level Snapshots in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Pool, Data Protection

Description: Learn how to create and manage pool-level snapshots in Ceph using rados, understand the difference between pool and image snapshots, and use snapshots for data recovery.

---

## Understanding Pool-Level Snapshots

Ceph supports two types of snapshots:
- **Pool snapshots** - snapshot of all objects in an entire pool at a point in time
- **Self-managed snapshots** - snapshots managed per-object (used by RBD and CephFS)

Pool snapshots are the simpler form and can be used directly with the RADOS object store. They capture a consistent view of all objects in the pool.

## Creating a Pool Snapshot

```bash
rados -p mypool mksnap my-snapshot-$(date +%Y%m%d)
```

List existing snapshots:

```bash
rados -p mypool lssnap
```

Output:
```
1   my-snapshot-20260331  2026.03.31 10:00:00
```

## Reading an Object from a Snapshot

Access object data at snapshot time:

```bash
rados -p mypool -s my-snapshot-20260331 get my-object /tmp/my-object-restored
```

## Listing Objects at Snapshot Time

```bash
rados -p mypool -s my-snapshot-20260331 ls
```

## Comparing Objects Between Snapshots

```bash
rados -p mypool -s snap-v1 get my-object /tmp/obj-v1
rados -p mypool -s snap-v2 get my-object /tmp/obj-v2
diff /tmp/obj-v1 /tmp/obj-v2
```

## Automating Pool Snapshots

Create a script for daily snapshots:

```bash
#!/bin/bash
POOL="mypool"
SNAP_NAME="daily-$(date +%Y%m%d-%H%M%S)"
RETENTION_DAYS=7

# Create snapshot
rados -p $POOL mksnap $SNAP_NAME
echo "Created snapshot: $SNAP_NAME"

# Clean up old snapshots beyond retention
rados -p $POOL lssnap | awk '{print $2}' | grep "^daily-" | sort | \
  head -n -$RETENTION_DAYS | while read snap; do
  echo "Removing old snapshot: $snap"
  rados -p $POOL rmsnap $snap
done
```

## Removing a Pool Snapshot

```bash
rados -p mypool rmsnap my-snapshot-20260331
```

## Important Limitations

Pool snapshots should NOT be used with RBD or CephFS pools:

- RBD pools use self-managed snapshots via the `rbd snap` command
- CephFS pools use MDS-managed snapshots
- Creating pool snapshots on RBD/CephFS pools can cause corruption

```bash
# Check if a pool is safe for pool-level snapshots
ceph osd dump | grep -A 5 "pool 'mypool'" | grep "pool_snaps\|flags"
```

## Pool Snapshots vs RBD Snapshots

| Feature | Pool Snapshot | RBD Image Snapshot |
|---------|--------------|-------------------|
| Scope | Entire pool | Single image |
| Cloning | Not supported | Supported |
| COW | Not supported | Supported |
| Rollback | Limited | Full |

## Summary

Pool-level snapshots in Ceph capture all objects in a RADOS pool at a specific point in time using `rados mksnap`. They are suitable for custom RADOS applications but should not be used with RBD or CephFS pools that manage their own snapshots. Use pool snapshots for simple object backup scenarios, automate them with scheduled scripts, and clean up old snapshots based on a retention policy.
