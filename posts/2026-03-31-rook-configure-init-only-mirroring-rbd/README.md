# How to Configure init-only Mirroring Mode for RBD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Mirroring, Snapshot

Description: Learn how to configure init-only mirroring mode for RBD images in Rook-Ceph for one-time initial sync without ongoing continuous replication.

---

## What Is init-only Mirroring Mode

The `init-only` mirroring mode is a special RBD snapshot mirroring mode introduced to handle the initial synchronization of large images to a secondary cluster. When set to `init-only`, the image is mirrored once (the full initial sync) and then stops replicating further changes.

This mode is useful for:
- Seeding a secondary cluster with a large base image before switching to continuous mirroring
- One-time data migration between clusters
- Creating a point-in-time copy on the secondary without ongoing overhead

## Step 1 - Enable Snapshot Mirroring on the Pool

The `init-only` mode requires snapshot-based mirroring to be configured at the pool level:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror pool enable replicapool image
```

## Step 2 - Enable init-only Mirroring on an Image

Enable `init-only` mode on the image to sync:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image enable replicapool/large-base-image init-only
```

## Step 3 - Create the Initial Snapshot

Trigger the first snapshot that will be synced to the secondary:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image snapshot replicapool/large-base-image
```

## Step 4 - Monitor the Initial Sync

Watch the sync progress on the secondary cluster:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror image status replicapool/large-base-image
```

```text
large-base-image:
  global_id:   xyz-789
  state:       up+syncing
  description: syncing, 45.2 GiB / 100 GiB
  last_update: 2026-03-31T10:05:00
```

Wait until the state changes to `up+stopped`:

```text
large-base-image:
  state:       up+stopped
  description: local image is primary
```

## Step 5 - Transition to Continuous Mirroring After Init

Once the initial sync is complete, transition to snapshot-based continuous mirroring:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image disable replicapool/large-base-image

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image enable replicapool/large-base-image snapshot

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image snapshot schedule add replicapool/large-base-image 1h
```

## Step 6 - Use init-only for Data Migration

For a one-time migration, complete the `init-only` sync, then use the secondary image as the new primary:

```bash
# Demote primary on source
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd mirror image demote replicapool/large-base-image

# Promote on secondary
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror image promote replicapool/large-base-image
```

## Step 7 - Verify the Final State

Confirm the image on the secondary is now primary and writable:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd mirror image status replicapool/large-base-image
```

```text
state: up+stopped
description: local image is primary
```

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph-secondary -- \
  rbd info replicapool/large-base-image | grep "mirroring primary"
```

```text
mirroring primary: true
```

## Summary

The RBD `init-only` mirroring mode provides an efficient way to perform the initial one-time synchronization of large images before enabling continuous replication. Enable it with `rbd mirror image enable <image> init-only`, trigger a snapshot, wait for the sync to complete, then transition to snapshot-based continuous mirroring or promote the secondary for a clean data migration. This avoids the overhead of continuous journal streaming during the potentially slow initial sync.
