# How to Configure Volume Replication via Rook CSI-Addons

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Csi-Addons, Volume Replication, Disaster Recovery, Kubernetes

Description: Learn how to configure RBD volume replication using Rook CSI-Addons to replicate PersistentVolumes to a secondary Ceph cluster for disaster recovery.

---

## Overview

Rook CSI-Addons provides volume replication capabilities through the `VolumeReplication` and `VolumeReplicationClass` CRDs. These enable asynchronous replication of RBD volumes from a primary Ceph cluster to a secondary cluster, supporting RPO-based disaster recovery for Kubernetes workloads.

## Prerequisites

Install CSI-Addons on both primary and secondary clusters:

```bash
kubectl apply -f https://github.com/csi-addons/kubernetes-csi-addons/releases/latest/download/crds.yaml
kubectl apply -f https://github.com/csi-addons/kubernetes-csi-addons/releases/latest/download/setup-controller.yaml
```

Both clusters must have RBD mirroring configured at the Ceph pool level.

## Step 1 - Enable RBD Pool Mirroring

Enable mirroring on the RBD pool in both clusters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  mirroring:
    enabled: true
    mode: image
    snapshotSchedules:
      - interval: 1h
        startTime: "00:00:00-05:00"
```

Get the bootstrap token from the primary cluster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool peer bootstrap create replicapool
```

Use the token on the secondary cluster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool peer bootstrap import replicapool <token>
```

## Step 2 - Create a VolumeReplicationClass

Define a VolumeReplicationClass on the primary cluster:

```yaml
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplicationClass
metadata:
  name: rook-volumereplicationclass
spec:
  provisioner: rook-ceph.rbd.csi.ceph.com
  parameters:
    mirroringMode: snapshot
    schedulingInterval: 1h
    schedulingStartTime: "00:00:00-05:00"
    replication.storage.openshift.io/replication-secret-name: rook-csi-rbd-provisioner
    replication.storage.openshift.io/replication-secret-namespace: rook-ceph
```

Apply the class:

```bash
kubectl apply -f volumereplicationclass.yaml
```

## Step 3 - Enable Replication for a PVC

Create a VolumeReplication resource to enable replication for a specific PVC:

```yaml
apiVersion: replication.storage.openshift.io/v1alpha1
kind: VolumeReplication
metadata:
  name: my-pvc-replication
  namespace: default
spec:
  volumeReplicationClass: rook-volumereplicationclass
  replicationState: primary
  dataSource:
    kind: PersistentVolumeClaim
    name: my-pvc
    apiGroup: ""
```

Apply the resource:

```bash
kubectl apply -f volume-replication.yaml
```

## Step 4 - Verify Replication Status

Check the VolumeReplication status:

```bash
kubectl get volumereplication my-pvc-replication -o yaml
```

Look for the status conditions:

```text
status:
  conditions:
  - type: Completed
    status: "True"
  - type: Degraded
    status: "False"
  - type: Resyncing
    status: "False"
  state: Primary
  message: "volume is marked primary"
```

Verify replication at the Ceph level:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image status replicapool/<image-name>
```

## Step 5 - Failover to Secondary

During a failover, promote the secondary volume to primary:

```yaml
spec:
  replicationState: secondary
```

Apply on the primary (if accessible):

```bash
kubectl patch volumereplication my-pvc-replication \
  --type merge \
  -p '{"spec":{"replicationState":"secondary"}}'
```

On the secondary cluster, promote the volume:

```yaml
spec:
  replicationState: primary
```

## Summary

Configuring volume replication via Rook CSI-Addons requires enabling RBD pool mirroring on both clusters, creating a VolumeReplicationClass with snapshot-based replication settings, and applying a VolumeReplication resource to individual PVCs. During a disaster, the secondary volume is promoted to primary, enabling workloads to resume on the secondary cluster with minimal data loss based on the configured snapshot interval.
