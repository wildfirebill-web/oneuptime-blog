# How to Configure Device Class Targeting for Block Pools in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Device Class, Block Pools, HDD, SSD, NVMe, CRUSH

Description: Learn how to configure device class targeting in Rook CephBlockPool to direct pool data to specific OSD device types like SSD or NVMe for performance tiering.

---

## What Are Device Classes in Ceph?

Ceph automatically classifies OSDs into device classes based on the underlying storage media:

- `hdd` - Hard disk drives
- `ssd` - Solid-state drives
- `nvme` - NVMe SSDs

Device classes allow you to create CRUSH rules that target specific hardware tiers. For example, a latency-sensitive database pool can target only NVMe OSDs, while an archive pool uses HDDs.

## Checking Current Device Classes

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# View device classes assigned to each OSD
ceph osd tree

# Or get a more detailed view
ceph device ls
ceph osd crush class ls
```

Example output from `ceph osd tree`:

```text
ID CLASS WEIGHT  TYPE NAME        STATUS REWEIGHT PRI-AFF
-1       8.00000 root default
-3       2.00000     host node-1
 0   ssd 0.50000         osd.0       up  1.00000 1.00000
 1   hdd 1.50000         osd.1       up  1.00000 1.00000
-5       2.00000     host node-2
 2   ssd 0.50000         osd.2       up  1.00000 1.00000
 3   hdd 1.50000         osd.3       up  1.00000 1.00000
```

## Configuring Device Class in CephBlockPool

Use the `deviceClass` field in the `CephBlockPool` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: ssd
  replicated:
    size: 3
```

This creates a CRUSH rule that only uses OSDs classified as `ssd`. Data written to this pool will never land on HDD OSDs.

## Creating Separate Pools for Different Tiers

```yaml
# High-performance NVMe pool for databases
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: nvme-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: nvme
  replicated:
    size: 3
---
# Standard SSD pool for general workloads
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: ssd
  replicated:
    size: 3
---
# HDD pool for archive/backup data
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: hdd-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: hdd
  replicated:
    size: 3
```

## Creating StorageClasses for Each Tier

```yaml
# NVMe StorageClass for databases
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-nvme
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: nvme-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# HDD StorageClass for archive data
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-hdd
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: hdd-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Manually Setting Device Classes

If Ceph auto-detects the wrong device class (common for cloud volumes or virtual disks), you can manually set it:

```bash
# List OSD device classes
ceph osd crush class ls

# Change OSD device class
ceph osd crush rm-device-class osd.0
ceph osd crush set-device-class ssd osd.0

# Verify
ceph osd tree | grep "osd.0"
```

## Verifying Device Class CRUSH Rule

After creating a pool with a device class, verify the CRUSH rule:

```bash
# Check pool's CRUSH rule
ceph osd pool get ssd-pool crush_rule

# Inspect the rule
ceph osd crush rule dump replicated_ssd_pool
```

Expected output showing `ssd` class restriction:

```text
{
    "rule_id": 2,
    "rule_name": "replicated_ssd_pool",
    "type": 1,
    "steps": [
        {
            "op": "take",
            "item_name": "default~ssd"   # ~ separates root and device class
        },
        ...
    ]
}
```

## Summary

Device class targeting in Rook `CephBlockPool` directs pool data to specific OSD hardware types (HDD, SSD, NVMe). Set the `deviceClass` field in the pool spec to create a CRUSH rule scoped to that device class. Create separate pools and StorageClasses for each hardware tier to give application teams named storage tiers to choose from. If auto-detection assigns incorrect classes (common with cloud block volumes), manually override with `ceph osd crush set-device-class`.
