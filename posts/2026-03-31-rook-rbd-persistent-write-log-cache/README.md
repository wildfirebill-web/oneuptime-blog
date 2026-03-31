# How to Configure RBD Persistent Write Log Cache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Cache, Performance

Description: Learn how to configure RBD persistent write log (PWL) cache in Ceph to accelerate write operations using local PMEM or NVMe storage.

---

## What Is RBD Persistent Write Log Cache

The RBD Persistent Write Log (PWL) cache is a write-back cache that stores incoming writes in fast local storage (PMEM or NVMe) before flushing them asynchronously to the Ceph cluster. This dramatically reduces write latency for I/O-sensitive workloads like databases.

Key benefits:
- Write acknowledgment from local cache instead of waiting for Ceph OSD replication
- Batching and coalescing of small random writes before sending to Ceph
- Crash safety: the write log is persistent, so in-flight writes survive a client crash

## Step 1 - Choose Cache Media

PWL cache supports two storage backends:

- **PMEM (Persistent Memory)** - Lowest latency, uses `rwl` mode
- **SSD/NVMe** - Higher latency than PMEM but still much faster than Ceph OSD RTT, uses `ssd` mode

For NVMe-based caching on Kubernetes nodes, use the `ssd` mode.

## Step 2 - Configure the Cache via Rook ConfigMap

Set the cache configuration using the Rook config override:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client]
    rbd_persistent_cache_mode = ssd
    rbd_persistent_cache_path = /mnt/nvme/rbd-pwl-cache
    rbd_persistent_cache_size = 21474836480
    rbd_cache = true
    rbd_cache_size = 67108864
    rbd_cache_max_dirty = 50331648
    rbd_cache_target_dirty = 33554432
```

## Step 3 - Prepare the Cache Directory on Each Node

Ensure the cache directory exists on the host with appropriate permissions:

```bash
mkdir -p /mnt/nvme/rbd-pwl-cache
chmod 700 /mnt/nvme/rbd-pwl-cache
```

Use a DaemonSet init container to prepare the directory:

```yaml
initContainers:
- name: prepare-cache
  image: busybox
  command: ["sh", "-c", "mkdir -p /mnt/nvme/rbd-pwl-cache && chmod 700 /mnt/nvme/rbd-pwl-cache"]
  securityContext:
    privileged: true
  volumeMounts:
  - name: nvme
    mountPath: /mnt/nvme
```

## Step 4 - Enable PWL on a Specific Image

Enable the write log cache feature on an RBD image:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd feature enable replicapool/myimage write-cache
```

## Step 5 - Verify Cache Activity

Check cache statistics for an active image:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd perf image iostat replicapool --format json
```

Look for `write_ops` being served from the cache before flushing to the cluster.

## Step 6 - Flush the Cache Manually

To force all dirty data to be written to Ceph before an operation like a snapshot:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd cache flush replicapool/myimage
```

## Step 7 - StorageClass Configuration

Configure the StorageClass to reference write log cache parameters:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-fast
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFeatures: layering,exclusive-lock,object-map,fast-diff,deep-flatten
  mounter: rbd
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
```

## Summary

RBD Persistent Write Log cache in Rook-Ceph reduces write latency by buffering writes to local PMEM or NVMe before flushing to the Ceph cluster. Configure it via the Rook config override ConfigMap, prepare cache directories on each node, and enable the `write-cache` feature on images that benefit most. Use `rbd cache flush` before creating snapshots to ensure consistency.
