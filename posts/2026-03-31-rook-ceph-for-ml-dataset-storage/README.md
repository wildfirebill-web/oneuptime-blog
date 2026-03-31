# How to Use Ceph for Machine Learning Dataset Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Machine Learning, Storage, Kubernetes

Description: Configure Rook-Ceph to serve as a scalable, high-performance storage backend for machine learning datasets in Kubernetes-based ML pipelines.

---

## Overview

Machine learning workloads require storage that handles large files, parallel reads, and multi-node data access. Ceph provides object storage (RGW), block storage (RBD), and filesystem (CephFS) interfaces suited to different ML data access patterns.

## Choosing the Right Interface

| Interface | Best For | Access Pattern |
|---|---|---|
| CephFS | Shared dataset directories | POSIX, multi-pod |
| RGW (S3) | Dataset versioning, remote access | S3 API |
| RBD | Single-node high-IOPS training | Block device |

## Setting Up a CephFS Dataset Volume

Create a dedicated CephFS filesystem for ML datasets:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: ml-datasets
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 2
    activeStandby: true
```

Create a StorageClass for dynamic provisioning:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-ml
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: ml-datasets
  pool: ml-datasets-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
```

## Mounting Datasets in Training Pods

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: imagenet-dataset
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs-ml
  resources:
    requests:
      storage: 500Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      containers:
        - name: trainer
          image: pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime
          volumeMounts:
            - name: dataset
              mountPath: /data/imagenet
      volumes:
        - name: dataset
          persistentVolumeClaim:
            claimName: imagenet-dataset
```

## Optimize for Sequential Reads

ML training reads large files sequentially. Tune the CephFS client:

```bash
# Set large read-ahead for CephFS mounts
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client readahead_max_bytes 67108864

# Increase objecter cache
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set client objecter_inflight_ops 24576
```

## Summary

Rook-Ceph is well suited for ML dataset storage, offering CephFS for shared multi-pod access with ReadWriteMany, S3-compatible RGW for versioned dataset management, and RBD for high-IOPS single-node training. Tuning readahead and objecter cache settings significantly improves sequential read throughput for large dataset files.
