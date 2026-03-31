# How to Recover from Accidental Pool Deletion in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Disaster Recovery, Pool Deletion, Recovery, Storage

Description: Learn how to recover data after accidental Ceph pool deletion, understand what data is recoverable, and implement safeguards to prevent future accidental deletions.

---

## Understanding Pool Deletion Consequences

When a Ceph pool is deleted, all objects in that pool are destroyed immediately and permanently. Unlike database tables, there is no built-in undo operation. Recovery requires either:
1. Restoring from a backup
2. Recovering from a snapshot taken before deletion
3. Reconstructing data from application-level backups

## Prevention: Protecting Pools from Deletion

The best recovery strategy is prevention. Ceph provides a nodelete flag:

```bash
# Prevent pool deletion
ceph osd pool set <pool-name> nodelete true

# Verify
ceph osd pool get <pool-name> nodelete
```

Rook also protects pools in the CephBlockPool spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```

Deleting a CephBlockPool CRD will not delete the pool if `ROOK_ALLOW_MULTIPLE_FILESYSTEMS` safeguards are enabled.

## Checking if Pool Deletion Protection Was Set

Check before deletion is attempted:

```bash
ceph osd pool ls detail | grep -E "nodelete|flags"
```

If the pool had `nodelete true`, the deletion would have been blocked:

```
Error EPERM: pool deletion is disabled; you must unset the nodelete flag for the pool first
```

## Recovering from Ceph Pool Snapshot (If Available)

If a pool-level snapshot was created before deletion, recovery is possible in some cases. Check for orphaned snapshots:

```bash
ceph osd pool ls
rados -p .pool-snapshots ls
```

For RBD images, check the trash:

```bash
rbd trash ls <pool-name>
rbd trash restore <pool-name>/<image-id>
```

## Recovering from RBD Snapshots

If RBD image snapshots were taken before deletion:

```bash
# List images in trash
rbd trash ls replicapool

# Restore from trash
rbd trash restore replicapool/<image-id>

# Roll back to a snapshot
rbd snap rollback replicapool/myimage@snap1
```

## Recovering from External Backup

If the pool was backed up externally, restore the pool and re-import:

```bash
# Recreate the pool
ceph osd pool create recovered-pool 32 32
ceph osd pool application enable recovered-pool rbd

# Import data from backup
rbd import /backup/image-backup.raw recovered-pool/restored-image
```

For CephFS, restore from backup:

```bash
# Recreate filesystem pools
ceph osd pool create cephfs-metadata 16
ceph osd pool create cephfs-data 32

# Recreate filesystem
ceph fs new cephfs cephfs-metadata cephfs-data
```

## Establishing Regular Pool Backups

Implement automated RBD export backups to prevent unrecoverable loss:

```bash
#!/bin/bash
POOL="replicapool"
BACKUP_DIR="/backup/rbd"
DATE=$(date +%Y-%m-%d)

for image in $(rbd ls $POOL); do
  rbd export $POOL/$image $BACKUP_DIR/$image-$DATE.raw
  echo "Backed up $image"
done
```

## Summary

Accidental Ceph pool deletion is generally unrecoverable without backups. The `nodelete` pool flag and Rook CRD safeguards prevent accidental deletion. If deletion has occurred, RBD trash recovery and external backups are the only paths to data restoration. Implementing automated RBD export backups and enabling `nodelete` on all critical pools is essential for data protection.
