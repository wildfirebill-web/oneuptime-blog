# How to Configure Longhorn Storage Classes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, StorageClass, Configuration

Description: Learn how to create and configure custom Longhorn StorageClasses to define different storage tiers and policies for your Kubernetes workloads.

## Introduction

Kubernetes StorageClasses allow administrators to define different storage tiers with varying performance characteristics, replication policies, and access modes. Longhorn supports rich StorageClass customization, enabling you to create multiple classes for different use cases — from high-availability production volumes to lightweight development storage.

## Understanding Longhorn StorageClass Parameters

Longhorn's StorageClass supports the following key parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `numberOfReplicas` | Number of volume replicas | `3` |
| `staleReplicaTimeout` | Timeout before marking replica stale (minutes) | `30` |
| `fromBackup` | Restore volume from a backup URL | `""` |
| `fsType` | Filesystem type (ext4, xfs) | `ext4` |
| `dataLocality` | Data locality preference (`disabled`, `best-effort`, `strict-local`) | `disabled` |
| `diskSelector` | Comma-separated list of disk tags to schedule replicas on | `""` |
| `nodeSelector` | Comma-separated list of node tags for scheduling | `""` |
| `recurringJobSelector` | Recurring job groups to associate with volumes | `""` |
| `backingImage` | Name of a backing image to use | `""` |

## The Default Longhorn StorageClass

After installation, Longhorn creates a default StorageClass:

```yaml
# View the default Longhorn StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
```

## Creating a High-Availability StorageClass

For production workloads that require maximum redundancy:

```yaml
# storage-class-ha.yaml - High-availability storage class with 3 replicas
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ha
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain  # Retain volumes when PVC is deleted
volumeBindingMode: Immediate
parameters:
  # Three replicas for high availability
  numberOfReplicas: "3"
  # Mark replica stale after 60 minutes of unavailability
  staleReplicaTimeout: "60"
  # Prefer storing a replica on the same node as the workload
  dataLocality: "best-effort"
  fsType: "ext4"
```

```bash
kubectl apply -f storage-class-ha.yaml
```

## Creating a Development StorageClass

For non-production environments where storage efficiency matters more than redundancy:

```yaml
# storage-class-dev.yaml - Single-replica storage for dev/test environments
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-dev
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete  # Automatically clean up volumes
volumeBindingMode: Immediate
parameters:
  # Single replica to save storage space
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
  fsType: "ext4"
```

```bash
kubectl apply -f storage-class-dev.yaml
```

## Creating a StorageClass with XFS Filesystem

Some workloads (e.g., databases like MongoDB) perform better with XFS:

```yaml
# storage-class-xfs.yaml - XFS filesystem storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-xfs
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  # Use XFS filesystem instead of the default ext4
  fsType: "xfs"
```

```bash
kubectl apply -f storage-class-xfs.yaml
```

## Creating a StorageClass with Disk and Node Selectors

Use tags to control which disks and nodes store replicas:

```yaml
# storage-class-ssd.yaml - Storage class targeting SSD-tagged disks
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ssd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  # Only schedule replicas on disks with the "ssd" tag
  diskSelector: "ssd"
  # Only schedule replicas on nodes with the "storage" tag
  nodeSelector: "storage"
  fsType: "ext4"
```

Before using disk/node selectors, tag your resources in Longhorn:

```bash
# Tag a node with the "storage" label via the Longhorn API
# Navigate to Longhorn UI → Node → Edit → Add Tag: "storage"
# Navigate to Longhorn UI → Node → Disk → Edit → Add Tag: "ssd"
```

## Creating a StorageClass with Strict Local Data Locality

For latency-sensitive workloads that require data to be on the same node:

```yaml
# storage-class-local.yaml - Strict local data locality
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-local
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "1"
  # Enforce that data is stored on the same node as the pod
  dataLocality: "strict-local"
  fsType: "ext4"
```

## Listing and Verifying StorageClasses

```bash
# List all storage classes in the cluster
kubectl get storageclass

# Describe a specific storage class
kubectl describe storageclass longhorn-ha

# Get the default storage class
kubectl get storageclass | grep "(default)"
```

## Setting a StorageClass as Default

```bash
# Remove default annotation from current default class
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'

# Set the new class as default
kubectl patch storageclass longhorn-ha \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

## Using a Custom StorageClass in a PVC

```yaml
# pvc-example.yaml - PVC using the high-availability storage class
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  # Reference the custom storage class
  storageClassName: longhorn-ha
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f pvc-example.yaml
kubectl get pvc app-data
```

## Conclusion

Longhorn's flexible StorageClass system allows you to define storage policies that match the requirements of different workloads. By creating multiple classes — from high-availability production tiers to lightweight development storage — you can optimize resource usage and ensure that each application gets the right storage characteristics. Always test new StorageClasses in a non-production environment before rolling them out to critical workloads.
