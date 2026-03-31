# How to Create a Ceph Backup Verification Runbook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Backup, Runbook, Verification, Kubernetes

Description: A backup verification runbook for Rook-Ceph covering RBD snapshot integrity checks, CephFS backup validation, RGW bucket replication verification, and restore testing.

---

## Why Backup Verification Matters

A backup that has never been tested is not a backup. This runbook provides systematic verification steps for all Ceph data paths: block storage (RBD), file storage (CephFS), and object storage (RGW).

## Step 1: Verify RBD Snapshots

List and verify RBD snapshots are being created:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd snap ls replicapool/my-volume

# Check snapshot size is not zero
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd snap ls replicapool/my-volume --format json | python3 -m json.tool
```

Test restoring an RBD snapshot:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd clone replicapool/my-volume@snap-20260101 replicapool/restore-test

# Mount and verify data integrity
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd map replicapool/restore-test
```

## Step 2: Verify Velero Backups

Check Velero backup status for the rook-ceph namespace:

```bash
velero backup get | grep rook
velero backup describe ceph-daily-backup-20260101 --details
```

Test a restore to a staging namespace:

```bash
velero restore create verify-restore \
  --from-backup ceph-daily-backup-20260101 \
  --namespace-mappings rook-ceph:rook-ceph-restore \
  --wait

kubectl -n rook-ceph-restore get pvc
kubectl -n rook-ceph-restore get cephcluster
```

## Step 3: Verify CephFS Backup

For CephFS backed by external tools like Restic or Kanister:

```bash
# Check Restic snapshots
restic -r s3:s3.amazonaws.com/backup-bucket/cephfs snapshots

# Verify latest snapshot integrity
restic -r s3:s3.amazonaws.com/backup-bucket/cephfs check

# Test restore of a subdirectory
restic -r s3:s3.amazonaws.com/backup-bucket/cephfs restore latest \
  --target /tmp/cephfs-verify \
  --include /data/critical-dir
```

## Step 4: Verify RGW Bucket Replication

If using multisite replication, verify sync status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync status

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket sync status --bucket critical-data
```

Spot-check object count parity between zones:

```bash
# Primary zone
aws s3 ls s3://critical-data --endpoint-url http://primary-rgw --recursive | wc -l

# Secondary zone
aws s3 ls s3://critical-data --endpoint-url http://secondary-rgw --recursive | wc -l
```

## Step 5: Checksum Verification

For critical data, verify checksum consistency after restore:

```bash
# Before backup (save checksums)
kubectl -n app exec -it app-pod -- find /data -type f -exec md5sum {} \; > /tmp/checksums-before.txt

# After restore
kubectl -n app exec -it app-pod-restored -- find /data -type f -exec md5sum {} \; > /tmp/checksums-after.txt

diff /tmp/checksums-before.txt /tmp/checksums-after.txt
```

## Summary

A Ceph backup verification runbook validates RBD snapshot integrity, Velero backup completeness, CephFS archive quality, and RGW replication parity. Regular restore testing - at minimum monthly - confirms that disaster recovery procedures will work when needed.
