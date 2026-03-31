# How to Set Up Ceph for Jupyter Notebook Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Jupyter, Storage, Kubernetes

Description: Configure Rook-Ceph to provide persistent, shared storage for JupyterHub deployments on Kubernetes, enabling notebook persistence and dataset sharing.

---

## Overview

JupyterHub deployments on Kubernetes need persistent storage for notebooks, datasets, and model files. Rook-Ceph provides per-user RWO volumes for notebooks and shared RWX volumes for common datasets.

## Architecture

- Per-user volumes: RBD (ReadWriteOnce) for private notebooks and user files
- Shared datasets: CephFS (ReadWriteMany) for common datasets
- Model artifacts: RGW (S3) for versioned model storage

## Create a StorageClass for User Notebooks

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-rbd-jupyter
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
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Configure JupyterHub with Rook Storage

In the JupyterHub Helm values:

```yaml
singleuser:
  storage:
    type: dynamic
    dynamic:
      storageClass: rook-ceph-rbd-jupyter
      pvcNameTemplate: claim-{username}
      volumeNameTemplate: volume-{username}
      storageAccessModes:
        - ReadWriteOnce
    capacity: 20Gi
  extraVolumes:
    - name: shared-datasets
      persistentVolumeClaim:
        claimName: shared-datasets-rwx
  extraVolumeMounts:
    - name: shared-datasets
      mountPath: /datasets
      readOnly: true
```

## Create Shared Dataset Volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-datasets-rwx
  namespace: jhub
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 1Ti
```

## Access S3 Storage from Notebooks

Configure notebook environment to access RGW:

```python
# In a Jupyter notebook cell
import boto3
import os

# These can be set via JupyterHub environment injection
s3 = boto3.client(
    's3',
    endpoint_url='http://rook-ceph-rgw-my-store.rook-ceph.svc',
    aws_access_key_id=os.environ.get('S3_ACCESS_KEY'),
    aws_secret_access_key=os.environ.get('S3_SECRET_KEY')
)

# List available model artifacts
response = s3.list_buckets()
for bucket in response['Buckets']:
    print(bucket['Name'])
```

## Enable Volume Expansion for Growing Notebooks

When user notebooks outgrow their initial 20Gi:

```bash
kubectl patch pvc claim-username -n jhub \
  --patch '{"spec": {"resources": {"requests": {"storage": "50Gi"}}}}'
```

## Summary

Rook-Ceph provides the ideal storage foundation for JupyterHub on Kubernetes. RBD provides per-user persistent notebook storage with ReclaimPolicy=Retain, CephFS enables shared read-only dataset volumes across all users, and RGW gives notebook users direct S3 API access to model artifacts and experiment results.
