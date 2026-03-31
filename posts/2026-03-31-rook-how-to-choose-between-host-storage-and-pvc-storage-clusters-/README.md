# How to Choose Between Host Storage and PVC Storage Clusters in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Host Storage, PVC Storage, Kubernetes, Architecture

Description: Compare host-based and PVC-based OSD storage models in Rook-Ceph to choose the right approach for your infrastructure and operational requirements.

---

## Two Models for OSD Storage in Rook

Rook-Ceph supports two fundamentally different ways to provide storage to Ceph OSDs:

1. **Host storage (direct device):** OSDs use raw block devices on the Kubernetes node's physical or virtual disks, configured via the `storage.nodes` or `storage.useAllNodes` section of the CephCluster CRD.

2. **PVC storage (storageClassDeviceSets):** OSDs use Kubernetes PersistentVolumeClaims backed by a StorageClass, configured via `storage.storageClassDeviceSets`.

## Host Storage Model

In the host storage model, Rook directly accesses block devices (e.g., `/dev/sdb`) on the node using host path mounts.

```yaml
spec:
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: "node-1"
        devices:
          - name: "sdb"
          - name: "sdc"
      - name: "node-2"
        devices:
          - name: "sdb"
```

**Advantages:**
- Maximum I/O performance - no virtualization overhead
- Full control over device selection
- Works without a pre-existing StorageClass

**Disadvantages:**
- Nodes must have raw, unused block devices available
- OSD pods are pinned to specific nodes; not portable
- Node decommissioning requires manual OSD removal first

## PVC Storage Model

In the PVC storage model, Rook creates PVCs for each OSD using a specified StorageClass, which can provision cloud volumes, local volumes, or any other CSI-backed storage.

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
                  storage: 500Gi
              storageClassName: gp3
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

**Advantages:**
- Works on cloud providers where raw devices are not available
- Supports `portable: true` for rescheduling OSD pods across nodes
- Scales by adjusting `count` without reconfiguring nodes
- Consistent with Kubernetes-native storage patterns

**Disadvantages:**
- Adds a layer of indirection (cloud volume or LVM-based storage beneath the PVC)
- May have additional cost for cloud block volumes
- Performance limited by the underlying StorageClass

## Decision Matrix

| Factor | Host Storage | PVC Storage |
|--------|-------------|-------------|
| Infrastructure | Bare metal / VMs with direct disks | Cloud / any Kubernetes cluster |
| Raw disk available | Yes | Not required |
| Portability | No (node-pinned) | Yes (with `portable: true`) |
| Performance | Highest | Depends on StorageClass |
| Scaling | Manual device addition | Increase `count` |
| Node replacement | Manual OSD drain needed | PVC follows pod (if portable) |

## Hybrid Deployments

You can use both models in the same cluster. For example, use host devices for the main high-performance pool and PVCs for archive or cold storage:

```yaml
spec:
  storage:
    nodes:
      - name: "fast-node-1"
        devices:
          - name: "nvme0n1"
    storageClassDeviceSets:
      - name: archive-set
        count: 6
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              storageClassName: standard-hdd
              volumeMode: Block
              resources:
                requests:
                  storage: 2Ti
              accessModes:
                - ReadWriteOnce
```

## Operational Considerations

**Node maintenance:**

With host storage, before draining a node, you must safely remove its OSDs:

```bash
# Mark OSDs on the node as out
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out osd.0 osd.1

# Drain the node after Ceph rebalances
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
```

With PVC storage and `portable: true`, the OSD pod can reschedule automatically to another node that can attach the PVC.

## Summary

Choose host storage for bare metal deployments where raw block devices are available and maximum I/O performance is required. Choose PVC storage for cloud or managed Kubernetes clusters where direct disk access is not possible, or where you need Kubernetes-native volume management and node portability. Hybrid configurations allow you to mix both models within a single Rook-Ceph cluster, using each where it is most appropriate.
