# How to Choose Between Host Storage and PVC Storage Clusters in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Storage Architecture, PVC, Bare Metal, Kubernetes

Description: Compare Rook-Ceph host-based storage (raw disks) vs PVC-based storage, and learn which to choose based on your environment and requirements.

---

## Two Storage Deployment Models

Rook-Ceph supports two primary OSD storage models:

1. **Host Storage** - OSDs use raw block devices directly attached to Kubernetes nodes
2. **PVC Storage** - OSDs use PersistentVolumeClaims backed by a StorageClass

The right choice depends on your Kubernetes environment, performance requirements, and operational preferences.

## Host Storage Model

In host storage mode, Rook discovers and uses raw block devices on Kubernetes nodes.

```yaml
spec:
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^sd[b-z]"
    # OR explicit node/device specification:
    nodes:
      - name: "node-1"
        devices:
          - name: "sdb"
          - name: "sdc"
```

### When to Use Host Storage

```text
- Bare metal Kubernetes clusters
- On-premises clusters with dedicated storage nodes
- When you need maximum I/O performance (no virtualization overhead)
- When nodes have pre-attached NVMe or SAS/SATA drives
- Production clusters with predictable, stable node infrastructure
```

### Host Storage Advantages

```text
+ Best I/O performance - no virtualization layer overhead
+ Lower cost - no cloud storage charges
+ Predictable latency - no shared storage contention
+ Simpler storage architecture - fewer moving parts
```

### Host Storage Disadvantages

```text
- Nodes must have raw (unused) disks attached
- OSDs are tied to specific nodes - node replacement is harder
- Disk failure requires physical replacement
- Node reimaging loses OSD data (use data disks, not OS disks)
```

## PVC Storage Model

In PVC storage mode, Rook creates OSDs backed by PersistentVolumeClaims:

```yaml
spec:
  storage:
    storageClassDeviceSets:
      - name: set1
        count: 3
        portable: true
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 100Gi
              storageClassName: gp2-block
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

### When to Use PVC Storage

```text
- Cloud Kubernetes environments (EKS, GKE, AKS)
- Environments with dynamically provisioned storage
- When nodes may be replaced, reimaged, or scaled in/out
- When you want flexible OSD placement (portable: true)
- When disaster recovery requires volume snapshots
```

### PVC Storage Advantages

```text
+ Works in cloud environments without raw disk access
+ OSDs can be rescheduled to different nodes (portable)
+ PVCs can be backed up with volume snapshots
+ Dynamic provisioning matches cloud-native patterns
+ Easier to scale: increase count in storageClassDeviceSets
```

### PVC Storage Disadvantages

```text
- Additional cost for cloud block storage
- Virtualization overhead adds latency
- Depends on an external storage provisioner
- More complex architecture (nested storage)
```

## Decision Matrix

```text
Factor                    Host Storage        PVC Storage
-------                   ------------        -----------
Environment               Bare metal          Cloud / on-prem with SAN
Node stability            Stable nodes        Nodes may be replaced
Performance priority      Maximum I/O         Cloud-native trade-offs
Cost structure            Hardware CAPEX      Cloud storage OPEX
Disaster recovery         Manual disk swap    PVC snapshots possible
OSD portability           Fixed to node       Portable across nodes
Scaling                   Add disks to nodes  Increase count parameter
```

## Hybrid Approach

You can combine both models in some scenarios - for example, using host storage for OSDs but PVC-backed monitors:

```yaml
spec:
  mon:
    count: 3
    volumeClaimTemplate:    # Monitors on PVC (portable, cloud-friendly)
      spec:
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
  storage:
    useAllNodes: false       # OSDs on host storage (performance)
    nodes:
      - name: "storage-1"
        devices:
          - name: "nvme0n1"
```

## Validating Your Choice

```bash
# For host storage - verify raw disks are clean
lsblk -d -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
# Disks to use must have no FSTYPE and no MOUNTPOINT

# For PVC storage - verify block StorageClass exists
kubectl get storageclass
kubectl describe storageclass gp2-block | grep VolumeBindingMode
```

## Summary

Choose host storage when running on bare metal with raw disks attached to nodes, where maximum I/O performance matters and node infrastructure is stable. Choose PVC storage in cloud environments, when nodes may be replaced, or when you need OSD portability. Both models are production-ready; the right choice is driven by your infrastructure type and operational requirements rather than technical capability.
