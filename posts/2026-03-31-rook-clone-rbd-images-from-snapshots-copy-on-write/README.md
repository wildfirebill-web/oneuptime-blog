# How to Clone RBD Images from Snapshots (Copy-on-Write)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Snapshot, Clone, Copy-on-Write, Kubernetes

Description: Learn how to create copy-on-write clones from RBD snapshots in Ceph, understand how clones share data with parent snapshots, and use clones for fast PVC provisioning.

---

## What is RBD Copy-on-Write Cloning?

RBD clone creates a new image that shares data with a parent snapshot using copy-on-write (COW). The clone is instantly created regardless of image size, and only divergent blocks are stored separately. Cloning is the foundation for fast PVC provisioning in Kubernetes and golden image workflows.

## Clone vs Snapshot vs Copy

| Feature | Snapshot | Clone | Full Copy |
|---------|----------|-------|-----------|
| Creation time | Instant | Instant | Minutes/Hours |
| Parent dependency | No | Yes | No |
| Independent writes | No (read-only) | Yes | Yes |
| Space on creation | Near zero | Near zero | Full size |

## Step 1: Create a Parent Snapshot

```bash
# Create the source image
rbd create mypool/golden-image --size 20G
rbd map mypool/golden-image
mkfs.ext4 /dev/rbd0
# ... install software, configure ...
rbd unmap /dev/rbd0

# Create a snapshot
rbd snap create mypool/golden-image@v1
```

## Step 2: Protect the Snapshot

You must protect a snapshot before creating clones from it:

```bash
rbd snap protect mypool/golden-image@v1
```

## Step 3: Create a Clone

```bash
rbd clone mypool/golden-image@v1 mypool/clone-for-app1
rbd clone mypool/golden-image@v1 mypool/clone-for-app2
```

Clones are instantly created regardless of size.

## Step 4: Verify the Clone

```bash
rbd info mypool/clone-for-app1
```

Output shows:
```
parent: mypool/golden-image@v1
overlap: 20 GiB
```

## Listing Clones of a Snapshot

```bash
rbd children mypool/golden-image@v1
```

Output:
```
mypool/clone-for-app1
mypool/clone-for-app2
```

## Using Clones in Kubernetes (CSI Provisioner)

When you restore a Kubernetes PVC from a VolumeSnapshot, the CSI driver creates an RBD clone under the hood:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app1-pvc
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: golden-image-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

The PVC is provisioned instantly because it is backed by a clone.

## Monitoring Clone Chain Depth

Deep clone chains (many generations of clones) can degrade performance. Check chain depth:

```bash
rbd info mypool/clone-for-app1 | grep "parent"
```

Flatten clones when chain depth exceeds 3-4 levels:

```bash
rbd flatten mypool/clone-for-app1
```

## Summary

RBD COW cloning provides instant image provisioning by creating images that share blocks with a parent snapshot. Protect the parent snapshot with `rbd snap protect` before cloning, then use `rbd clone` to create independent writable images. In Kubernetes, PVC restoration from VolumeSnapshots leverages this mechanism for fast volume provisioning. Monitor and flatten clone chains exceeding 3-4 levels to maintain read performance.
