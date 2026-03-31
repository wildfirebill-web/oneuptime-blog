# How to Implement ReadWriteOnce (RWO) Storage with Ceph RBD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Kubernetes, Storage, PVC, Access Mode, Block Storage

Description: Configure ReadWriteOnce persistent volumes using Ceph RBD block storage for Kubernetes workloads that require exclusive single-node read/write access to block devices.

---

## What is ReadWriteOnce?

ReadWriteOnce (RWO) is a Kubernetes PersistentVolume access mode that allows a volume to be mounted as read-write by a single node at a time. It is the most common access mode for databases, stateful applications, and any workload requiring exclusive block device access.

Ceph RBD (RADOS Block Device) is the ideal backend for RWO volumes because RBD images are block devices that map to a single node efficiently via the kernel RBD driver or the Ceph CSI driver.

## StorageClass Configuration

First, ensure you have a StorageClass using the Ceph CSI RBD provisioner:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Creating an RWO PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-database-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 50Gi
```

## Using the PVC in a StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: my-database-pvc
```

## Verifying RWO Attachment

Check that the PVC is bound and the volume is attached to one node:

```bash
kubectl get pvc my-database-pvc -n production
kubectl describe pvc my-database-pvc -n production | grep -E "Access|Node|Status"

# Check the RBD image on the Ceph side
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  rbd info replicapool/csi-vol-<uuid>
```

## RWO Limitations and Considerations

RWO volumes cannot be attached to multiple nodes simultaneously. If a Pod is rescheduled to a different node:

1. The old node must release the volume first (requires clean Pod termination)
2. The kubelet on the new node attaches the RBD image
3. The filesystem is mounted on the new node

To speed up failover, set `terminationGracePeriodSeconds` appropriately on your Pod and consider using Pod Disruption Budgets.

## Summary

ReadWriteOnce PVCs backed by Ceph RBD provide durable, performant block storage for databases and stateful Kubernetes workloads. The Ceph CSI driver handles RBD image lifecycle management automatically, and the StorageClass configuration is straightforward. For workloads requiring multi-node access, consider CephFS with ReadWriteMany instead.
