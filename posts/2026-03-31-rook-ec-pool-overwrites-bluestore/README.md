# How to Enable Overwrites for Erasure Coded Pools (BlueStore Required)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, BlueStore, Storage

Description: Learn how to enable the allow_ec_overwrites flag on Ceph erasure coded pools to support RBD and CephFS workloads that require in-place data modification.

---

By default, erasure coded pools in Ceph only support append-only workloads. Writing to an existing object requires reading the full stripe, decoding it, modifying the relevant chunk, re-encoding, and writing back - a process known as a read-modify-write (RMW) cycle. Enabling overwrites allows Ceph to perform this RMW cycle, but it requires BlueStore as the OSD backend.

## Why Overwrites Are Disabled by Default

Erasure coding was originally designed for object storage where objects are written once and read many times. Allowing overwrites introduces the RMW cycle, which:

- Adds latency on every write to an existing object
- Requires partial stripe reads before writes
- Demands a reliable OSD backend that handles partial writes atomically

BlueStore, the modern Ceph OSD backend, supports this via its transactional write model, making overwrite support safe.

## Checking Your OSD Backend

Before enabling overwrites, confirm all OSDs backing the pool use BlueStore:

```bash
ceph osd metadata | python3 -c "
import json, sys
osds = json.load(sys.stdin)
for osd in osds:
    print(osd.get('id'), osd.get('osd_objectstore'))
"
```

All entries should show `bluestore`. Filestore OSDs do not support EC overwrites.

## Enabling Overwrites on a Pool

```bash
ceph osd pool set ec-pool allow_ec_overwrites true
```

Verify the setting:

```bash
ceph osd pool get ec-pool allow_ec_overwrites
```

Expected output:

```text
allow_ec_overwrites: true
```

## Using EC Pools with RBD (Block Storage)

Once overwrites are enabled, you can create an RBD image on the EC data pool combined with a replicated metadata pool:

```bash
# Create replicated pool for RBD metadata
ceph osd pool create rbd-meta 32 32 replicated

# Create EC pool for RBD data
ceph osd pool create rbd-ec-data 64 64 erasure ec-profile

# Enable overwrites on the EC data pool
ceph osd pool set rbd-ec-data allow_ec_overwrites true

# Enable RBD application on both pools
ceph osd pool application enable rbd-meta rbd
ceph osd pool application enable rbd-ec-data rbd

# Create RBD image using both pools
rbd create --size 10G --data-pool rbd-ec-data rbd-meta/my-ec-image
```

## Rook CephBlockPool with Overwrites

In Rook, enable overwrites via pool parameters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-data-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    allow_ec_overwrites: "true"
```

## Performance Implications

Enabling overwrites on an EC pool means every overwrite of existing data triggers an RMW cycle. This is significantly slower than replicated pools for random overwrite workloads:

```text
Operation         Replicated Pool   EC Pool (k=4,m=2)
Sequential Write  1x                0.9x (aligned)
Random Overwrite  1x                0.3-0.5x (RMW penalty)
Read              1x                1.0x (no penalty)
```

Use EC pools with overwrites for workloads that are predominantly sequential or read-heavy, such as database backups, video files, or archive data accessed via RBD.

## Summary

The `allow_ec_overwrites` flag unlocks EC pool use with RBD and CephFS data pools, but requires all backing OSDs to use BlueStore. The read-modify-write overhead makes EC overwrites unsuitable for random write-heavy workloads. Reserve this configuration for large-block sequential or archive workloads where the storage efficiency benefit of erasure coding outweighs the write performance penalty.
