# How to Migrate Data Between Ceph Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Storage, Data Migration, Pool

Description: Step-by-step guide to migrating data between Ceph pools using rados cppool, rbd migration, and export-import techniques without downtime.

---

## Why Migrate Data Between Ceph Pools

You might need to migrate data between Ceph pools for several reasons:
- Moving from a replicated pool to an erasure-coded pool for storage efficiency
- Migrating data to a pool with different CRUSH rules (e.g., SSD vs HDD)
- Consolidating multiple pools into one
- Changing the replication factor of stored data

## Method 1: Migrating RADOS Objects with rados cppool

The simplest method for migrating all objects from one pool to another:

```bash
# Create the destination pool first
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool create new-pool 128 128

# Copy all objects from source to destination
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados cppool source-pool new-pool
```

Monitor progress:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados df | grep -E "new-pool|source-pool"
```

Output:

```text
POOL_NAME    USED  OBJECTS  CLONES  COPIES  MISSING_ON_PRIMARY  UNFOUND  DEGRADED  RD_OPS  RD  WR_OPS  WR  USED COMPR  UNDER COMPR
source-pool  100G    50000       0  150000                   0        0         0    1000  10G    2000  20G           0            0
new-pool     100G    50000       0  150000                   0        0         0       0   0B       0   0B           0            0
```

## Method 2: Migrating RBD Images

For block storage images, use the `rbd migration` feature for live, non-disruptive migration:

```bash
# Prepare the migration - this is safe to do on a live image
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd migration prepare \
  source-pool/my-image \
  dest-pool/my-image

# Check migration status
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd migration status source-pool/my-image
```

Output:

```text
{
    "source_spec": "{\"pool_name\":\"source-pool\",\"pool_id\":5,\"image_name\":\"my-image\"}",
    "destination_spec": "{\"pool_name\":\"dest-pool\",\"pool_id\":8,\"image_name\":\"my-image\"}",
    "state": "prepared",
    "state_description": ""
}
```

Execute the migration (data is actually copied):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd migration execute source-pool/my-image
```

Commit the migration when complete:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd migration commit source-pool/my-image
```

## Method 3: Copying Individual RADOS Objects

To migrate specific objects rather than entire pools:

```bash
# Copy a single object
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados -p source-pool cp my-object dest-pool/my-object

# Verify the copy
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados -p dest-pool stat my-object
```

## Method 4: Using rbd export/import for Offline Migration

For images that can tolerate downtime:

```bash
# Export from source pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd export source-pool/my-image - | \
  rbd import - dest-pool/my-image

# Or export to a file first
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd export source-pool/my-image /tmp/image-backup.raw
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd import /tmp/image-backup.raw dest-pool/my-image
```

## Migrating Kubernetes PVCs Between Pools

To migrate a Kubernetes PVC from one StorageClass to another:

```bash
#!/bin/bash
# migrate-pvc.sh - Migrate PVC data to a new storage class

SOURCE_PVC="my-pvc"
DEST_PVC="my-pvc-new"
NAMESPACE="my-app"
NEW_STORAGE_CLASS="rook-ceph-block-ssd"
SIZE="50Gi"

# Create new PVC
kubectl -n $NAMESPACE apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: $DEST_PVC
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: $SIZE
  storageClassName: $NEW_STORAGE_CLASS
EOF

# Run a migration pod
kubectl -n $NAMESPACE apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pvc-migrator
spec:
  containers:
  - name: migrator
    image: alpine
    command: ["sh", "-c", "cp -av /source/. /dest/ && echo DONE"]
    volumeMounts:
    - name: source
      mountPath: /source
    - name: dest
      mountPath: /dest
  volumes:
  - name: source
    persistentVolumeClaim:
      claimName: $SOURCE_PVC
  - name: dest
    persistentVolumeClaim:
      claimName: $DEST_PVC
  restartPolicy: Never
EOF
```

## Verifying Migration Integrity

After migration, verify data integrity:

```bash
# For RADOS pools - compare object counts
SRC_COUNT=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados -p source-pool ls | wc -l)
DST_COUNT=$(kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rados -p dest-pool ls | wc -l)
echo "Source: $SRC_COUNT objects, Dest: $DST_COUNT objects"

# For RBD images - compare checksums
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd export source-pool/my-image - | md5sum
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- rbd export dest-pool/my-image - | md5sum
```

## Summary

Ceph provides multiple strategies for migrating data between pools depending on your use case. Use `rados cppool` for bulk pool-to-pool migration, `rbd migration` for live non-disruptive RBD image migration, or `rbd export/import` for offline migrations. Always verify data integrity after migration before decommissioning the source pool.
