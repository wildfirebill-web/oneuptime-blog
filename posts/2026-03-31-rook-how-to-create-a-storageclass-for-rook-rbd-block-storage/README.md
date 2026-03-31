# How to Create a StorageClass for Rook RBD Block Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, StorageClass, Kubernetes, Block Storage

Description: Learn how to create and configure a Kubernetes StorageClass for Rook-Ceph RBD block storage, including CSI parameters, volume binding, and reclaim policies.

---

## Overview

Rook integrates with Kubernetes dynamic provisioning through StorageClasses. An RBD (RADOS Block Device) StorageClass allows Kubernetes to automatically create Ceph block volumes when PersistentVolumeClaims are created. This is the most common storage type for databases and stateful applications.

## Prerequisites

Before creating the StorageClass, you need:

- A working Rook-Ceph cluster
- A `CephBlockPool` already created
- The Rook CSI driver running in your cluster

## Create a CephBlockPool

First, create the block pool that will back the StorageClass:

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

Apply it:

```bash
kubectl apply -f blockpool.yaml
```

## Create the StorageClass

Now create the StorageClass referencing the pool:

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
mountOptions:
  - discard
```

Apply the StorageClass:

```bash
kubectl apply -f storageclass-rbd.yaml
```

## Set as Default StorageClass (Optional)

To make this the default StorageClass for your cluster:

```bash
kubectl patch storageclass rook-ceph-block \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Test with a PersistentVolumeClaim

Create a PVC to verify the StorageClass works:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc-test
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```

Apply and check the status:

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc rbd-pvc-test
```

The PVC should reach the `Bound` state:

```text
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES
rbd-pvc-test   Bound    pvc-abc12345-6789-0000-abcd-ef0123456789   1Gi        RWO
```

## Verify the RBD Image

Check that the RBD image was created in the Ceph pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls replicapool
```

## Summary

Creating a Rook RBD StorageClass involves two steps: defining a `CephBlockPool` for the failure domain and replication factor, then creating a `StorageClass` that points to that pool using the Rook CSI provisioner. Once in place, any PVC referencing this StorageClass automatically provisions a Ceph block volume. Enable `allowVolumeExpansion: true` to support online volume resizing.
