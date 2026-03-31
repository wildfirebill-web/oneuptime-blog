# How to Configure Dynamic Storage Provisioning Policies in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Dynamic Provisioning, Storage Class, Policy, Kubernetes

Description: Configure dynamic storage provisioning policies in Rook-Ceph using StorageClasses, VolumeBindingMode, and reclaim policies to control how PVCs are fulfilled.

---

## Dynamic Provisioning Overview

Rook-Ceph's CSI driver enables dynamic PVC provisioning - volumes are created on-demand when a PVC is submitted. Provisioning policies in StorageClass resources control volume size, performance tier, reclaim behavior, and binding timing.

## Default StorageClass Configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
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
volumeBindingMode: Immediate
```

## Volume Binding Mode Policies

```yaml
# Immediate: PV created when PVC is submitted (default)
volumeBindingMode: Immediate

# WaitForFirstConsumer: PV created only when a pod needs it
# Useful for topology-aware provisioning
volumeBindingMode: WaitForFirstConsumer
```

## Reclaim Policy Options

```yaml
# Delete: PV deleted when PVC is deleted (dev/test environments)
reclaimPolicy: Delete

# Retain: PV kept after PVC deletion (production data)
reclaimPolicy: Retain

# For production storage classes, always use Retain
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-retain
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Mount Options for Performance Tuning

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-optimized
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  csi.storage.k8s.io/fstype: ext4
mountOptions:
  - discard       # Enable TRIM for SSDs
  - noatime       # Reduce metadata write overhead
  - nodiratime
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Topology-Aware Provisioning

```yaml
# Limit provisioning to specific zones/nodes
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-zone-a
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: zone-a-pool
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - zone-a
```

## Set Provisioning Limits via ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-limits
  namespace: production
spec:
  hard:
    requests.storage: "5Ti"
    rook-ceph-block.storageclass.storage.k8s.io/requests.storage: "2Ti"
    persistentvolumeclaims: "100"
```

## Summary

Dynamic provisioning policies in Rook-Ceph are configured through StorageClass parameters, volumeBindingMode, reclaimPolicy, and mountOptions. Using `Retain` reclaim policy for production and `WaitForFirstConsumer` binding for topology-sensitive workloads ensures data safety and optimal volume placement across the cluster.
