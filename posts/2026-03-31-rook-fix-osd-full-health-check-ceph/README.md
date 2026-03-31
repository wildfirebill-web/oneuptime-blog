# How to Fix OSD_FULL Health Check in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, OSD, Capacity, Emergency

Description: Learn how to urgently resolve the OSD_FULL health error in Ceph when an OSD exceeds its full threshold and write I/O is halted to prevent data corruption.

---

## Understanding OSD_FULL

`OSD_FULL` is a critical health error that fires when one or more OSDs exceed the `full_ratio` threshold (default 0.95 or 95% utilized). When an OSD hits full, Ceph stops all write I/O to pools mapped to that OSD to prevent data corruption. This effectively makes the cluster read-only for affected pools.

Check current health:

```bash
ceph health detail
```

Example output:

```text
HEALTH_ERR 1 full osd(s)
[ERR] OSD_FULL: 1 osd(s) are full
    osd.5 is full (97.8% >= 95%)
```

## Immediate Assessment

Check which OSDs are full:

```bash
ceph osd df | awk '$8 > 90 {print}'
```

Check affected pools:

```bash
ceph df detail
ceph pg stat
```

Determine which pools map to the full OSD using its CRUSH weight:

```bash
ceph osd tree
ceph osd map <pool-name> <object-name>
```

## Emergency Option 1: Temporarily Raise the Full Ratio

Buy time to fix the underlying capacity issue:

```bash
ceph osd set-full-ratio 0.97
```

This restores write I/O but should only be a stopgap. Do not raise above 0.99.

## Emergency Option 2: Delete Data

If data can be deleted to free space immediately:

```bash
# Identify large objects
rados -p <pool-name> ls | head -20
rados -p <pool-name> stat <object-name>

# Delete unnecessary objects
rados -p <pool-name> rm <object-name>

# Delete an entire bucket (RGW)
radosgw-admin bucket rm --bucket=<bucket-name> --purge-objects
```

## Adding More OSD Capacity

The permanent fix is to add more storage capacity:

In Rook, add more OSDs by expanding the storage configuration:

```yaml
spec:
  storage:
    storageClassDeviceSets:
    - name: set1
      count: 6  # increase from current count
      volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          resources:
            requests:
              storage: 1Ti
          storageClassName: fast-ssd
          volumeMode: Block
          accessModes:
          - ReadWriteOnce
```

Apply:

```bash
kubectl -n rook-ceph apply -f cephcluster.yaml
```

## Rebalancing After Adding Capacity

After adding OSDs, trigger rebalancing to redistribute data:

```bash
ceph osd reweight-by-utilization
```

Monitor rebalancing progress:

```bash
ceph -w | grep -E "backfill|recover|rebalance"
```

## Moving Data to a Less Full Pool

If only certain pools are filling up, enable pool compression:

```bash
ceph osd pool set <pool-name> compression_mode aggressive
ceph osd pool set <pool-name> compression_algorithm zstd
```

Or migrate data to erasure coded pools for better efficiency:

```bash
ceph osd pool create ec-pool 32 32 erasure
ceph osd pool set ec-pool allow_ec_overwrites true
```

## Summary

`OSD_FULL` stops write I/O when OSDs hit the full threshold. Immediately raise the full ratio slightly to restore writes, then fix the root cause: delete unnecessary data, add more OSD capacity, or enable compression on busy pools. After adding capacity, run `ceph osd reweight-by-utilization` to spread data across the expanded cluster. Monitor OSD utilization continuously to prevent this emergency from recurring.
