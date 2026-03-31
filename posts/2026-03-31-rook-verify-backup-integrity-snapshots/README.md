# How to Verify Backup Integrity from Ceph Snapshots

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Backup, Data Integrity

Description: Learn how to verify that Ceph snapshots and exported backups are complete and uncorrupted using checksum validation and test restoration procedures.

---

Taking snapshots is only half of a backup strategy. Verifying that those snapshots are complete and can be successfully restored is what makes a backup actually useful. Ceph provides tools to validate snapshot integrity at multiple levels.

## Creating a Snapshot for Testing

Create a pool snapshot as a baseline:

```bash
rados -p mypool mksnap mysnap
rados -p mypool lssnap
```

For RBD volumes (block storage):

```bash
rbd snap create mypool/myimage@mysnap
rbd snap ls mypool/myimage
```

For CephFS:

```bash
mkdir /mnt/cephfs/.snap/mysnap
ls /mnt/cephfs/.snap/
```

## Computing Checksums Before Backup

Before exporting, record checksums of critical objects:

```bash
#!/bin/bash
POOL="mypool"
CHECKSUMS="/tmp/checksums-$(date +%Y%m%d).txt"

for obj in $(rados ls -p $POOL); do
  rados get -p $POOL $obj /tmp/obj_tmp
  echo "$(md5sum /tmp/obj_tmp | awk '{print $1}') $obj" >> $CHECKSUMS
done
echo "Checksums written to $CHECKSUMS"
```

## Exporting an RBD Snapshot

Export a snapshot to a file:

```bash
rbd export mypool/myimage@mysnap /backup/myimage-snap.img
```

Compute checksum of the export:

```bash
sha256sum /backup/myimage-snap.img > /backup/myimage-snap.img.sha256
```

## Verifying the Export Integrity

After the export completes:

```bash
sha256sum -c /backup/myimage-snap.img.sha256
```

Expected output:

```
/backup/myimage-snap.img: OK
```

## Test Restoration to Verify Completeness

Import the exported image to a test pool:

```bash
rbd import /backup/myimage-snap.img testpool/restored-image

# Map the restored image
rbd map testpool/restored-image

# Mount and verify filesystem
mount /dev/rbd0 /mnt/test-restore
ls -la /mnt/test-restore
fsck /dev/rbd0
```

## Verifying CephFS Snapshot Consistency

Mount the snapshot and diff against source:

```bash
diff -r /mnt/cephfs/.snap/mysnap /mnt/cephfs/current-data
```

Or use rsync in dry-run mode:

```bash
rsync -anv /mnt/cephfs/.snap/mysnap/ /mnt/verify/
```

## Automated Verification CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: verify-ceph-backup
  namespace: rook-ceph
spec:
  schedule: "0 4 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: verify
            image: ceph/ceph:latest
            command:
            - /bin/bash
            - -c
            - |
              rbd export testpool/myimage@latest /tmp/verify.img
              sha256sum /tmp/verify.img
          restartPolicy: OnFailure
```

## Summary

Verifying backup integrity from Ceph snapshots requires checksumming objects before and after export, performing test restorations, and running filesystem checks on restored volumes. Automating this process via Kubernetes CronJobs ensures ongoing confidence that your backups are usable, not just present.
