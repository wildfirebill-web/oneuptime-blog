# How to Perform RBD Live Migration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Migration, Block Storage

Description: Learn how to use RBD live migration to move block device images between pools or formats in Ceph without downtime in Rook-Ceph deployments.

---

## What Is RBD Live Migration

RBD live migration allows you to move an RBD image from a source location to a destination while the image remains accessible and writable. This is useful for:

- Migrating images from one pool to another (e.g., from HDD to NVMe pool)
- Upgrading from RBD format 1 to format 2
- Moving images between Ceph clusters
- Changing the data layout or replication settings

Live migration works by lazily copying objects from source to destination as they are accessed, while writing new data exclusively to the destination.

## Step 1 - Prepare the Destination Pool

Ensure the destination pool exists and is configured:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  ceph osd pool create fast-pool 32 replicated

kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd pool init fast-pool
```

## Step 2 - Start the Live Migration

Prepare the migration (source image must be unmounted or in read-only mode for prep):

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration prepare replicapool/myimage fast-pool/myimage
```

Check migration status:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration status replicapool/myimage
```

Sample output:

```text
source: replicapool/myimage
destination: fast-pool/myimage
state: prepared
```

## Step 3 - Execute the Migration

Start the background copy process:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration execute fast-pool/myimage
```

Monitor progress:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration status fast-pool/myimage
```

```text
state: executing
source: replicapool/myimage
destination: fast-pool/myimage
executed: 65%
```

## Step 4 - Commit the Migration

Once execution reaches 100%, commit to finalize the migration and remove the source:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration commit fast-pool/myimage
```

After commit, the source image no longer exists and the destination is the sole copy.

## Step 5 - Abort a Migration

If you need to cancel before committing, abort the migration:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- \
  rbd migration abort fast-pool/myimage
```

This restores the source image to its original state.

## Step 6 - Update PV References in Kubernetes

After migration, update the PersistentVolume to reference the new pool:

```bash
kubectl patch pv <pv-name> -p '{"spec":{"csi":{"volumeAttributes":{"pool":"fast-pool"}}}}'
```

Then delete and recreate the PVC binding, or use a Velero backup-restore cycle for a cleaner migration.

## Summary

RBD live migration in Rook-Ceph enables zero-downtime movement of block images between pools using a prepare-execute-commit workflow. The image remains accessible throughout, with lazy object copying ensuring minimal impact. After migrating, update Kubernetes PersistentVolume references to point to the new pool location to complete the transition.
