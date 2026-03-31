# How to Migrate Applications Using RBD Mirroring in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Mirroring, Migration

Description: Learn how to use Rook RBD mirroring to migrate stateful Kubernetes applications from a primary cluster to a secondary cluster with minimal downtime.

---

## Overview

RBD mirroring in Rook enables live replication of block volumes between clusters. This makes it possible to migrate stateful applications - databases, message queues, and other workloads with persistent volumes - from a source cluster to a destination cluster with minimal downtime. The migration process involves promoting the secondary image, reconfiguring applications to use the new volume, and optionally reversing the mirror direction.

## Prerequisites

- Both clusters have RBD mirroring configured and the mirror daemon running
- The target pool has mirroring enabled and images are in `up+replaying` state on the secondary
- Network connectivity exists between both clusters

## Step 1 - Verify Replication is Healthy

Before starting migration, confirm replication lag is minimal:

```bash
# On secondary cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror pool status replicapool --verbose

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image status replicapool/<image-name>
```

Wait until `entries_behind_primary: 0` or `replay_lag: 0s` before proceeding.

## Step 2 - Stop Writes to the Primary Application

For a clean migration, stop the application on the primary cluster to prevent write divergence:

```bash
# Scale down the application
kubectl scale deployment my-app --replicas=0
# OR delete the pod
kubectl delete pod my-app-0

# Verify the PVC is not mounted
kubectl get pvc my-app-data
```

## Step 3 - Promote the Secondary Image

On the secondary cluster, promote the RBD image from secondary to primary:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image promote replicapool/<image-name>
```

For a forced promotion (if primary is unreachable):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image promote --force replicapool/<image-name>
```

## Step 4 - Create a PersistentVolume on the Secondary Cluster

Create a static PV that references the promoted RBD image:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: migrated-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  rbd:
    monitors:
    - 192.168.2.1:6789
    pool: replicapool
    image: <image-name>
    user: admin
    secretRef:
      name: ceph-secret
  persistentVolumeReclaimPolicy: Retain
  storageClassName: rook-ceph-block
```

```bash
kubectl apply -f migrated-pv.yaml
```

## Step 5 - Deploy the Application on the Secondary Cluster

Deploy the application on the secondary cluster using the migrated PV:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  volumeName: migrated-pv
  storageClassName: rook-ceph-block
```

```bash
kubectl apply -f migrated-pvc.yaml
kubectl apply -f my-app-deployment.yaml
```

## Step 6 - Demote the Primary Image (Optional)

After the application is running on the secondary, demote the original primary image to prevent dual-primary conflicts:

```bash
# On primary cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image demote replicapool/<image-name>
```

## Step 7 - Reverse the Mirror Direction (Optional)

If you want to use the new secondary as the ongoing primary and replicate back:

```bash
# Enable reverse replication on the original primary cluster
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd mirror image enable replicapool/<image-name> snapshot
```

## Summary

Migrating applications using RBD mirroring in Rook involves stopping writes, waiting for replication to catch up, promoting the secondary image, and deploying the application on the destination cluster. For minimal downtime, automate steps 2 through 5 with a migration script that can be executed in sequence. Always verify data integrity on the promoted image before routing production traffic to the migrated application.
