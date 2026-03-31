# How to Configure CephFS for Distributed ML Training Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Machine Learning, Distributed Training

Description: Configure CephFS in Rook to support distributed multi-node ML training, enabling simultaneous high-throughput data access across GPU worker pods.

---

## Overview

Distributed ML training frameworks like PyTorch DDP and Horovod require all worker nodes to access the same dataset simultaneously. CephFS with ReadWriteMany PVCs enables this pattern while Ceph handles data distribution.

## Deploy a High-Throughput CephFS Filesystem

Create a CephFS filesystem optimized for ML training:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: ml-training-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
    parameters:
      compression_mode: none
  dataPools:
    - name: ec-data
      erasureCoded:
        dataChunks: 4
        codingChunks: 2
      parameters:
        compression_mode: none
  metadataServer:
    activeCount: 3
    activeStandby: true
    resources:
      limits:
        cpu: "4"
        memory: "8Gi"
```

## Create a ReadWriteMany PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data-rwx
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs-ml
  resources:
    requests:
      storage: 2Ti
```

## Deploy Distributed Training Job

Use a Kubernetes Job with multiple parallel workers:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pytorch-ddp-training
spec:
  completions: 4
  parallelism: 4
  template:
    spec:
      containers:
        - name: trainer
          image: pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime
          command:
            - python
            - -m
            - torch.distributed.launch
            - --nproc_per_node=1
            - train.py
          env:
            - name: DATA_PATH
              value: /mnt/training-data
          volumeMounts:
            - name: training-data
              mountPath: /mnt/training-data
          resources:
            limits:
              nvidia.com/gpu: "1"
      volumes:
        - name: training-data
          persistentVolumeClaim:
            claimName: training-data-rwx
```

## Tune CephFS for Parallel Read Workloads

Configure the MDS for high concurrency:

```bash
# Increase MDS cache size for large directory listings
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 8589934592

# Enable async directory operations
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph config set mds mds_log_max_segments 128
```

Set client-side mount options in the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-ml
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: ml-training-fs
  pool: ml-training-fs-ec-data
  kernelMountOptions: "rsize=1048576,wsize=1048576,noatime"
```

## Summary

CephFS with ReadWriteMany PVCs enables distributed ML training by allowing all worker pods to read the same dataset simultaneously. Deploying multiple active MDS daemons, tuning MDS cache size, and setting optimal kernel mount options significantly improves throughput for parallel workloads running across GPU nodes.
