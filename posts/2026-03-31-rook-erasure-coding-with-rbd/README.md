# How to Use Erasure Coding with RBD in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, ErasureCoding, RBD, Storage

Description: Learn how to configure Ceph RBD block storage using an erasure coded data pool combined with a replicated metadata pool to save storage while maintaining block device support.

---

Ceph RBD (RADOS Block Device) natively supports erasure coded data pools combined with a replicated metadata pool. This hybrid approach lets you use storage-efficient erasure coding for the bulk of block data while keeping metadata operations fast and reliable on a replicated pool.

## Architecture Overview

An EC-backed RBD setup uses two pools:

- **Metadata pool** (replicated): stores the RBD image header, object map, parent chain, and other small metadata objects
- **Data pool** (erasure coded): stores the actual block data chunks, which are the vast majority of storage consumption

```text
RBD Image
   |
   +--> Metadata Pool (3-way replicated, small, fast)
   |
   +--> Data Pool (EC k=4,m=2, large, efficient)
```

## Step 1: Create the Erasure Code Profile

```bash
ceph osd erasure-code-profile set rbd-ec-profile \
  plugin=jerasure \
  technique=reed_sol_van \
  k=4 \
  m=2 \
  crush_failure_domain=host
```

## Step 2: Create Both Pools

```bash
# Replicated metadata pool
ceph osd pool create rbd-metadata 32 32 replicated
ceph osd pool application enable rbd-metadata rbd

# Erasure coded data pool
ceph osd pool create rbd-ec-data 64 64 erasure rbd-ec-profile
ceph osd pool set rbd-ec-data allow_ec_overwrites true
ceph osd pool application enable rbd-ec-data rbd
```

## Step 3: Create an RBD Image Using Both Pools

```bash
rbd create \
  --size 100G \
  --data-pool rbd-ec-data \
  rbd-metadata/my-ec-volume
```

Verify the image configuration:

```bash
rbd info rbd-metadata/my-ec-volume
```

Output will show:

```text
rbd image 'my-ec-volume':
        size 100 GiB in 25600 objects
        ...
        data_pool: rbd-ec-data
```

## Step 4: Rook CephBlockPool Configuration

In Rook, configure the EC data pool and a companion StorageClass:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: rbd-ec-data
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    allow_ec_overwrites: "true"
---
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: rbd-metadata
  namespace: rook-ceph
spec:
  replicated:
    size: 3
```

## Step 5: StorageClass for EC-Backed RBD

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-ec
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: rbd-metadata
  dataPool: rbd-ec-data
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Performance Considerations

EC-backed RBD performs well for large sequential I/O but has higher latency for random small writes due to the RMW cycle. Benchmark your workload before committing to EC for latency-sensitive applications like databases. For archival volumes, virtual machine images stored offline, or backup targets, EC RBD offers significant storage savings.

## Summary

Using erasure coding with RBD requires a replicated metadata pool and an EC data pool with `allow_ec_overwrites` enabled. Rook supports this via separate CephBlockPool resources and a StorageClass referencing both. This configuration can cut raw storage consumption by up to 50% compared to 3-way replication for block devices used by sequential or archival workloads.
