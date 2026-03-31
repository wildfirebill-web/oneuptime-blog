# How to Optimize RBD Cache Policy in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Performance, Cache, Storage

Description: Configure RBD client-side cache policies in Ceph to improve block device read and write performance by tuning cache size, writeback behavior, and flush intervals.

---

## Overview of RBD Client Cache

Ceph RBD (RADOS Block Device) supports client-side caching that buffers reads and writes before committing to the cluster. Two modes exist: `writeback` (fastest, acknowledges writes before flushing) and `writethrough` (safer, waits for OSD acknowledgment). The right cache policy depends on your workload's durability vs. performance requirements.

## Checking Current Cache Settings

Inspect the active librbd configuration:

```bash
ceph config dump | grep rbd
rbd info <pool>/<image> | grep features
```

View default cache parameters:

```bash
ceph config get client rbd_cache
ceph config get client rbd_cache_size
ceph config get client rbd_cache_max_dirty
```

## Key Cache Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rbd_cache` | true | Enable/disable client cache |
| `rbd_cache_size` | 32 MiB | Total cache size per image |
| `rbd_cache_max_dirty` | 24 MiB | Max dirty (unflushed) data |
| `rbd_cache_target_dirty` | 16 MiB | Target dirty data level |
| `rbd_cache_max_dirty_age` | 1.0s | Max age before flush |
| `rbd_cache_writethrough_until_flush` | true | Writethru until first flush |

## Configuring Writeback Cache

Enable writeback mode for maximum write performance:

```bash
ceph config set client rbd_cache true
ceph config set client rbd_cache_size 67108864        # 64 MiB
ceph config set client rbd_cache_max_dirty 50331648   # 48 MiB
ceph config set client rbd_cache_target_dirty 33554432 # 32 MiB
ceph config set client rbd_cache_max_dirty_age 2.0
ceph config set client rbd_cache_writethrough_until_flush false
```

## Configuring via ceph.conf on Nodes

For non-Kubernetes environments, add to `/etc/ceph/ceph.conf`:

```ini
[client]
rbd cache = true
rbd cache size = 67108864
rbd cache max dirty = 50331648
rbd cache target dirty = 33554432
rbd cache max dirty age = 2.0
rbd cache writethrough until flush = false
```

## Configuring in Rook StorageClass

Set RBD cache parameters in the StorageClass for Kubernetes workloads:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Pass librbd options via the CSI configmap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-csi-config
  namespace: rook-ceph
data:
  config.json: |
    [
      {
        "clusterID": "rook-ceph",
        "monitors": ["mon1:6789","mon2:6789","mon3:6789"],
        "rbd": {
          "radosNamespace": ""
        }
      }
    ]
```

## Disabling Cache for Latency-Sensitive Workloads

Some workloads (databases with their own buffer pools) benefit from bypassing RBD cache entirely:

```bash
ceph config set client rbd_cache false
```

Or disable per-image using image features:

```bash
rbd config image set <pool>/<image> rbd_cache false
```

## Monitoring Cache Effectiveness

Track cache hits and misses:

```bash
ceph daemon client.<id> perf dump | python3 -m json.tool | grep librbd
# Look for: librbd.cache_reads, librbd.cache_read_hits
```

## Summary

RBD cache tuning balances write performance against durability. Writeback mode with a 64 MiB cache significantly improves write throughput by coalescing small writes before flushing to OSDs. For Kubernetes workloads on Rook, configure cache parameters via the CSI driver configuration or the global Ceph config, and disable caching for applications that manage their own I/O buffers.
