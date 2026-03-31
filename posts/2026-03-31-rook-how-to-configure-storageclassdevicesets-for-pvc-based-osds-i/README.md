# How to Configure StorageClassDeviceSets for PVC-Based OSDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, StorageClassDeviceSets, PVC, OSD, Kubernetes

Description: Learn how to configure StorageClassDeviceSets in the Rook CephCluster CRD to provision PVC-backed OSDs, including count, portability, and volume templates.

---

## What Are StorageClassDeviceSets

`StorageClassDeviceSets` is the mechanism in the Rook `CephCluster` CRD for creating OSDs backed by PersistentVolumeClaims rather than raw host disks. Each "set" defines a group of OSDs that share the same StorageClass and configuration.

This is the primary OSD provisioning method for cloud environments where physical disk access is not available.

## Basic StorageClassDeviceSets Configuration

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    storageClassDeviceSets:
      - name: set1
        count: 3
        portable: true
        encrypted: false
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              resources:
                requests:
                  storage: 200Gi
              storageClassName: gp2-block
              volumeMode: Block
              accessModes:
                - ReadWriteOnce
```

## Key Fields Explained

### count

The number of OSD pods to create in this set. Each OSD gets its own PVC.

```yaml
count: 6    # Creates 6 OSDs, each with a 200Gi PVC = 1.2Ti total raw storage
```

To scale up, increase `count` and apply - Rook creates new OSD pods for the new PVCs.

### portable

Controls whether OSDs can be rescheduled to different nodes:

```yaml
portable: true   # OSD can run on any node that can claim its PVC
portable: false  # OSD is tied to the node where its PVC was first created
```

Use `portable: true` for cloud environments with autoscaling or node replacements.

### encrypted

Enables Ceph-level disk encryption for OSD data:

```yaml
encrypted: true   # Requires a KMS or Vault configuration
encrypted: false  # Default - no encryption
```

See the Rook encryption documentation when enabling this option.

## Multiple Device Sets

You can define multiple sets with different configurations:

```yaml
storage:
  storageClassDeviceSets:
    - name: ssd-osds
      count: 3
      portable: true
      volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            resources:
              requests:
                storage: 500Gi
            storageClassName: fast-ssd-block   # NVMe/SSD
            volumeMode: Block
            accessModes:
              - ReadWriteOnce

    - name: hdd-osds
      count: 6
      portable: true
      volumeClaimTemplates:
        - metadata:
            name: data
          spec:
            resources:
              requests:
                storage: 2Ti
            storageClassName: standard-hdd-block  # HDD for capacity
            volumeMode: Block
            accessModes:
              - ReadWriteOnce
```

## Separate WAL and Metadata Devices

For better performance, separate the OSD write-ahead log (WAL) and metadata (DB) onto faster storage:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data      # Main data volume (large, slower)
    spec:
      resources:
        requests:
          storage: 1Ti
      storageClassName: hdd-block
      volumeMode: Block
      accessModes:
        - ReadWriteOnce

  - metadata:
      name: metadata  # RocksDB metadata on fast storage
    spec:
      resources:
        requests:
          storage: 20Gi
      storageClassName: nvme-block
      volumeMode: Block
      accessModes:
        - ReadWriteOnce

  - metadata:
      name: wal       # Write-ahead log on fastest storage
    spec:
      resources:
        requests:
          storage: 10Gi
      storageClassName: nvme-block
      volumeMode: Block
      accessModes:
        - ReadWriteOnce
```

## Placement and Scheduling

Control which nodes can host OSDs from a specific set:

```yaml
- name: set1
  count: 3
  portable: true
  placement:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: ceph-storage
                operator: In
                values:
                  - "true"
    tolerations:
      - key: storage
        operator: Exists
        effect: NoSchedule
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: rook-ceph-osd
```

## Monitoring PVC-Based OSDs

```bash
# Check PVC status for OSD set
kubectl -n rook-ceph get pvc | grep set1

# Check OSD pod to PVC mapping
kubectl -n rook-ceph get pods -l app=rook-ceph-osd -o wide

# Verify Ceph sees all OSDs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

```text
ID  CLASS  WEIGHT   TYPE NAME
-1         1.7998   root default
-3         0.5999       host node-aaa
 0    ssd  0.1999           osd.0
-4         0.5999       host node-bbb
 1    ssd  0.1999           osd.1
-5         0.5999       host node-ccc
 2    ssd  0.1999           osd.2
```

## Scaling OSD Count

To add more OSDs, increase `count` and apply:

```bash
# Edit the CephCluster
kubectl -n rook-ceph edit cephcluster rook-ceph

# Change count: 3 to count: 6

# Rook will create 3 new PVCs and OSD pods
kubectl -n rook-ceph get pvc -w
```

## Summary

`StorageClassDeviceSets` enables PVC-based OSD provisioning in Rook-Ceph without requiring raw disk access. Each set defines a count of OSDs backed by a specific StorageClass, with `portable: true` for cloud environments. Add multiple sets to use different storage classes for tiered storage. Use separate WAL and metadata volume templates to boost performance by placing RocksDB metadata on faster storage than data. Scale OSDs by increasing the `count` field and reapplying the CephCluster manifest.
