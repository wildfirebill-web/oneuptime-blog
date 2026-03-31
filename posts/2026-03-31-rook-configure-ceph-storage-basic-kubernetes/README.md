# How to Configure Ceph Storage for a Basic Kubernetes Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Kubernetes, Storage, StorageClass

Description: Configure Ceph RBD and CephFS storage for a basic Kubernetes cluster using Rook, including StorageClass, PVC provisioning, and access mode configuration.

---

Rook makes it straightforward to add Ceph-backed persistent storage to a Kubernetes cluster. This guide walks through the minimal configuration needed to provision PVCs backed by Ceph RBD and CephFS.

## Prerequisites

A running Rook-Ceph operator and CephCluster:

```bash
kubectl get pods -n rook-ceph | grep -E "operator|mgr|mon|osd"
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- ceph status
```

## Creating an RBD Pool

RBD (RADOS Block Device) is the block storage backend for ReadWriteOnce PVCs:

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

```bash
kubectl apply -f blockpool.yaml
kubectl get cephblockpool -n rook-ceph
```

## Creating a StorageClass for RBD

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

```bash
kubectl apply -f storageclass-rbd.yaml
kubectl get sc rook-ceph-block
```

## Creating a CephFS Filesystem for Shared Storage

CephFS supports ReadWriteMany, needed for multiple pods sharing one volume:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

```bash
kubectl apply -f filesystem.yaml
```

## Provisioning a PVC

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
      storage: 10Gi
  storageClassName: rook-ceph-block
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc my-app-data
# Should show STATUS: Bound
```

## Verifying Storage Works

```bash
# Mount the PVC in a test pod and write data
kubectl run test-pod --image=busybox \
  --overrides='{"spec":{"containers":[{"name":"test","image":"busybox","command":["sh","-c","echo hello > /data/test.txt && cat /data/test.txt"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"my-app-data"}}]}}' \
  --restart=Never

kubectl logs test-pod
```

## Summary

Configuring Ceph storage for Kubernetes requires creating a CephBlockPool, a StorageClass pointing to that pool, and optionally a CephFilesystem for shared storage. Once the StorageClass is in place, PVCs are provisioned automatically by Rook's CSI driver. Testing PVC creation and data writes immediately after setup confirms that the full provisioning path is working.
