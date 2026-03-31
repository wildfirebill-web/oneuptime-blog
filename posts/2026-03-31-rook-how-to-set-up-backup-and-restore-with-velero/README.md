# How to Set Up Backup and Restore with Velero for Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Velero, Backup, Restore, Kubernetes, Disaster Recovery

Description: Learn how to configure Velero with the Ceph CSI snapshot plugin to back up and restore Kubernetes workloads using Rook-Ceph volume snapshots.

---

## Overview

Velero is a Kubernetes backup and restore tool that integrates with the CSI snapshot API to create volume snapshots as part of application backups. When used with Rook-Ceph, Velero can back up both Kubernetes resource manifests and persistent volume data using Ceph's native snapshot capability.

## Prerequisites

- A running Rook-Ceph cluster
- The CSI snapshot controller installed
- An S3-compatible bucket for Velero's backup storage (can be your Rook object store)

## Step 1 - Install the CSI Snapshot Controller

Install the snapshot CRDs and controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/deploy/kubernetes/snapshot-controller/
```

## Step 2 - Create a VolumeSnapshotClass for Rook

Create a snapshot class for RBD volumes:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbdplugin-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: rook-ceph.rbd.csi.ceph.com
deletionPolicy: Delete
parameters:
  clusterID: rook-ceph
  csi.storage.k8s.io/volumesnapshot/secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/volumesnapshot/secret-namespace: rook-ceph
```

Apply the snapshot class:

```bash
kubectl apply -f snapshotclass.yaml
```

## Step 3 - Configure the Rook Object Store as Backup Target

Get credentials for a Rook object store user to use as backup storage:

```bash
ACCESS_KEY=$(kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-velero-user \
  -o jsonpath='{.data.AccessKey}' | base64 --decode)
SECRET_KEY=$(kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-velero-user \
  -o jsonpath='{.data.SecretKey}' | base64 --decode)
RGW_ENDPOINT=$(kubectl -n rook-ceph get svc rook-ceph-rgw-my-store \
  -o jsonpath='{.spec.clusterIP}')
```

Create a credentials file:

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id=${ACCESS_KEY}
aws_secret_access_key=${SECRET_KEY}
EOF
```

## Step 4 - Install Velero

Install Velero with the AWS plugin (compatible with S3-compatible stores) and CSI plugin:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0,velero/velero-plugin-for-csi:v0.7.0 \
  --bucket velero-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true \
  --features=EnableCSI \
  --backup-location-config \
    region=us-east-1,s3ForcePathStyle="true",s3Url=http://${RGW_ENDPOINT}:80
```

## Step 5 - Create a Backup

Back up an entire namespace including PVCs:

```bash
velero backup create my-app-backup \
  --include-namespaces my-app \
  --snapshot-volumes=true \
  --volume-snapshot-locations default
```

Monitor backup progress:

```bash
velero backup describe my-app-backup --details
```

## Step 6 - Restore from Backup

Restore the namespace to the same or a different cluster:

```bash
velero restore create my-app-restore \
  --from-backup my-app-backup
```

Monitor restore progress:

```bash
velero restore describe my-app-restore
```

## Scheduling Automated Backups

Create a scheduled backup that runs daily:

```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces my-app \
  --snapshot-volumes=true \
  --ttl 720h
```

## Summary

Setting up Velero for Rook-Ceph backup involves installing the CSI snapshot controller, creating a VolumeSnapshotClass labeled for Velero, configuring Velero with the AWS-compatible plugin pointing to the Rook object store, and enabling the CSI feature flag. Once configured, Velero coordinates application-consistent backups that include both Kubernetes resource manifests and Ceph volume snapshots, enabling full application restores.
