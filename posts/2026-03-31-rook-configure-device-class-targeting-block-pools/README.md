# How to Configure Device Class Targeting for Block Pools in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Device Class, Block Pool, CRUSH, Storage Tier, Kubernetes

Description: Configure device class targeting in Rook-Ceph block pools to route data to specific OSD hardware tiers such as NVMe, SSD, or HDD for performance isolation.

---

## Why Target a Specific Device Class

In a Ceph cluster with mixed storage hardware, all OSDs participate in the same CRUSH map by default, and data may land on any OSD regardless of disk type. Device class targeting restricts a pool's CRUSH rule to a specific hardware tier so that block volumes always use the intended storage class. This enables cost-effective tiered storage where latency-sensitive databases use NVMe OSDs and bulk archive workloads use HDD OSDs.

## Device Classes in Ceph

Ceph recognizes three default device classes:

- `hdd` - Spinning hard disk drives
- `ssd` - Solid-state drives (SATA or SAS SSD)
- `nvme` - NVMe drives

You can also define custom classes. Ceph auto-detects the class when an OSD is created based on the rotational flag from the kernel block device.

## Confirming OSD Class Assignments

Before creating a targeted pool, verify that OSDs have the expected classes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd df tree | grep -E "(osd|CLASS|hdd|ssd|nvme)"
```

Or list all OSDs with their classes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush tree --show-shadow | grep -E "(hdd|ssd|nvme)"
```

## Setting deviceClass on a CephBlockPool

The `deviceClass` field in the CephBlockPool spec causes Rook to generate a CRUSH rule that restricts placement to OSDs of that class:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: nvme-block-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: nvme
  replicated:
    size: 3
```

For an HDD-only archive pool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: hdd-archive-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  deviceClass: hdd
  replicated:
    size: 3
```

## Verifying the Generated CRUSH Rule

After applying the manifest, Rook creates a CRUSH rule named after the pool that includes a filter on the device class. Inspect it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rule dump nvme-block-pool
```

The rule output contains a `step set_chooseleaf_tries` and a class filter restricting OSD selection to `nvme` devices. If OSDs of the requested class do not exist, the pool will have unhealthy PGs.

## Creating Multiple Tiered StorageClasses

Create a separate StorageClass for each device class pool so workloads can choose the appropriate tier via `storageClassName`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-nvme
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: nvme-block-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-hdd
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: hdd-archive-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Minimum OSD Count per Class

With `failureDomain: host` and `size: 3`, you need at least three hosts that each have at least one OSD of the targeted class. If you target `nvme` but only two hosts have NVMe drives, the pool will report `undersized` PGs. Check:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph health detail
```

## Summary

Device class targeting in Rook-Ceph block pools routes data to specific OSD hardware tiers by generating a CRUSH rule filtered to a single device class. Set `deviceClass: nvme`, `ssd`, or `hdd` on the CephBlockPool spec, ensure enough OSDs of that class exist across the required number of failure domain buckets, and expose the pool through a dedicated StorageClass so workloads can select the right tier at provisioning time.
