# How to Migrate Data Between Replicated and Erasure Coded Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, Migration, Erasure Coding, Kubernetes, Storage

Description: Learn how to safely migrate data between replicated and erasure coded Ceph pools to optimize storage efficiency or performance characteristics.

---

## Why Migrate Between Pool Types

Replicated pools offer the lowest latency and support all Ceph features, while erasure coded (EC) pools provide better storage efficiency (typically 1.5x overhead vs 3x for 3-way replication). You might want to migrate data from a replicated pool to an EC pool as data ages and access patterns shift, or vice versa for data that becomes more actively accessed.

## Important Limitation: No In-Place Migration

Ceph does not support directly converting a replicated pool to an erasure coded pool or vice versa. Data must be copied between pools. Plan for the migration to take time proportional to the amount of data being moved.

## Step 1 - Create the Destination Pool

Create the target erasure coded pool:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ec-pool
  namespace: rook-ceph
spec:
  erasureCoded:
    dataChunks: 4
    codingChunks: 2
  parameters:
    pg_autoscale_mode: "on"
    allow_ec_overwrites: "true"
```

```bash
kubectl apply -f ec-pool.yaml
```

## Step 2 - Export Data from the Source Pool

For RBD images, export each image to a file or directly to the destination:

```bash
# Export RBD image from replicated pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd export replicated-pool/my-image /tmp/my-image.raw

# Import into EC pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd import /tmp/my-image.raw ec-pool/my-image
```

## Step 3 - Direct Pool-to-Pool Copy with rados cppool

For object pools (used by RGW or CephFS), use `rados cppool`:

```bash
# Copy all objects from one pool to another
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados cppool source-pool destination-pool
```

Note: `cppool` copies metadata but you should verify object integrity afterward.

## Step 4 - Migrate RBD Volumes Live with rbd migration

For live migrations with minimal downtime:

```bash
# Prepare the migration
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd migration prepare \
  replicated-pool/source-image \
  ec-pool/dest-image

# Execute the migration (background copy)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd migration execute ec-pool/dest-image

# Commit once migration is complete
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd migration commit ec-pool/dest-image
```

## Step 5 - Update Storage Class and PVC References

After migration, update your StorageClass to point to the new pool and recreate PVCs as needed, or use Kubernetes volume cloning if supported.

## Verify Data Integrity

```bash
# Compare object counts
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p source-pool ls | wc -l

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p destination-pool ls | wc -l
```

## Summary

Migrating data between replicated and erasure coded Ceph pools requires explicit data copying since in-place conversion is not supported. For RBD images, use `rbd migration` for live migrations with minimal downtime. For object data, use `rados cppool`. Always verify data integrity after migration and update StorageClass references before decommissioning the source pool.
