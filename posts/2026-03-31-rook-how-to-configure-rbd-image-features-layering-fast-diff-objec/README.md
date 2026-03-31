# How to Configure RBD Image Features (Layering, Fast-Diff, Object-Map) in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Image Features, Kubernetes, Block Storage

Description: Learn which RBD image features to enable in Rook StorageClasses and how Layering, Fast-Diff, and Object-Map affect performance and compatibility.

---

## Overview

Ceph RBD images support a set of optional features that enhance functionality but require kernel or librbd support. When provisioning volumes through Rook, you select which features to enable via the `imageFeatures` parameter in the StorageClass. Choosing the right combination affects performance, snapshot behavior, and compatibility with older kernels.

## Common RBD Image Features

| Feature | Description |
|---------|-------------|
| `layering` | Enables copy-on-write cloning from snapshots |
| `fast-diff` | Speeds up `rbd diff` operations for incremental backups |
| `object-map` | Tracks which objects exist to speed up diff and export |
| `deep-flatten` | Removes parent references from clones recursively |
| `exclusive-lock` | Ensures only one client has write access at a time |
| `journaling` | Enables write-ahead logging for mirroring |

## Setting Image Features in a StorageClass

Configure `imageFeatures` as a comma-separated list in the StorageClass parameters:

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
  imageFeatures: layering,fast-diff,object-map,deep-flatten,exclusive-lock
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Recommended Feature Sets

### Full-Featured (Ceph CSI Default)

For clusters using the Ceph CSI driver with modern kernels:

```text
layering,fast-diff,object-map,deep-flatten,exclusive-lock
```

### Minimal for Older Kernels

If your nodes run older kernel versions (before 4.14), use only:

```text
layering
```

### Mirroring-Enabled

For RBD mirroring (disaster recovery scenarios):

```text
layering,exclusive-lock,journaling
```

Note that `journaling` requires `exclusive-lock` to be enabled as well.

## Verifying Image Features on an Existing Volume

Use the Ceph toolbox to inspect features on a specific RBD image:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd info replicapool/csi-vol-abc12345-6789
```

The output shows enabled features:

```text
rbd image 'csi-vol-abc12345-6789':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: abc12345def6
        block_name_prefix: rbd_data.abc12345def6
        format: 2
        features: layering, fast-diff, object-map, deep-flatten, exclusive-lock
```

## Enabling or Disabling Features on Existing Images

You can modify features on an existing image (unmount first):

```bash
# Enable a feature
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd feature enable replicapool/csi-vol-abc12345-6789 fast-diff

# Disable a feature
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd feature disable replicapool/csi-vol-abc12345-6789 journaling
```

## Summary

RBD image features in Rook are configured via the `imageFeatures` field in the StorageClass. The recommended default set includes `layering`, `fast-diff`, `object-map`, `deep-flatten`, and `exclusive-lock`, which gives the best performance and snapshot capabilities on modern kernels. For older kernels or mirroring scenarios, adjust the feature set accordingly - and always verify compatibility before deploying to production.
