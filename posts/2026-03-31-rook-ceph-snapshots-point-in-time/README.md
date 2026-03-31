# How to Set Up Ceph Snapshots for Point-in-Time Backups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Snapshot, Backup, RBD, CephFS, Point-in-Time

Description: Learn how to configure and manage Ceph RBD and CephFS snapshots for point-in-time backups, including scheduling, retention, and restoration procedures.

---

## Overview

Ceph snapshots capture the state of a block device or filesystem at a specific point in time with minimal overhead, using copy-on-write mechanics. This guide covers setting up point-in-time backups using both RBD and CephFS snapshots in a Rook-Ceph cluster.

## RBD Snapshots for Block Storage

### Create an RBD Snapshot

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# List existing images
rbd ls replicapool

# Create a snapshot
rbd snap create replicapool/myimage@snap-2026-03-31

# List snapshots
rbd snap ls replicapool/myimage
```

### Protect and Clone from Snapshot

```bash
# Protect the snapshot before cloning
rbd snap protect replicapool/myimage@snap-2026-03-31

# Clone from the snapshot
rbd clone replicapool/myimage@snap-2026-03-31 replicapool/myimage-backup
```

## CephFS Snapshots for File Storage

### Enable Snapshots on CephFS

Create a snapshot directory in a CephFS volume:

```bash
kubectl exec -it deploy/rook-ceph-tools -n rook-ceph -- bash
ceph fs ls

# Create a snapshot via the .snap directory
mkdir /mnt/cephfs/myvolume/.snap/snap-2026-03-31
```

### Use Kubernetes VolumeSnapshots

Create a VolumeSnapshotClass:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
driver: rook-ceph.rbd.csi.ceph.com
deletionPolicy: Retain
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph
```

Create a VolumeSnapshot:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: pvc-snapshot-2026-03-31
  namespace: production
spec:
  volumeSnapshotClassName: csi-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: my-pvc
```

```bash
kubectl apply -f snapshot.yaml
kubectl get volumesnapshot -n production
```

## Restore from a Snapshot

Create a new PVC from the snapshot:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-restored
  namespace: production
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: pvc-snapshot-2026-03-31
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Automate Snapshots with a CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rbd-snapshot-job
  namespace: rook-ceph
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshot
            image: rook/ceph:v1.13.0
            command:
            - /bin/bash
            - -c
            - |
              SNAP="snap-$(date +%Y%m%d-%H%M)"
              rbd snap create replicapool/myimage@$SNAP
              # Prune snapshots older than 7 days
              rbd snap ls replicapool/myimage --format json | \
                jq -r '.[].name' | head -n -28 | \
                xargs -I{} rbd snap rm replicapool/myimage@{}
          restartPolicy: OnFailure
```

## Summary

Ceph snapshots provide efficient point-in-time recovery with minimal storage overhead thanks to copy-on-write mechanics. Using the Kubernetes VolumeSnapshot API with Rook-Ceph CSI drivers integrates snapshot management natively into your cluster workflows. Automating snapshot creation and retention with CronJobs ensures consistent recovery points without manual intervention.
