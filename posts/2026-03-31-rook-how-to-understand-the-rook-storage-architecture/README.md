# How to Understand the Rook Storage Architecture (Operator, CSI, Daemon Layers)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Architecture, Kubernetes, CSI, Storage Operator

Description: A comprehensive overview of the Rook-Ceph architecture, explaining the operator, CSI driver, and Ceph daemon layers and how they interact in Kubernetes.

---

## Rook Architecture Overview

Rook is a Kubernetes operator that automates the deployment, management, and scaling of Ceph storage. The architecture has three primary layers:

1. **Rook Operator** - the Kubernetes controller that manages Ceph cluster lifecycle
2. **Ceph CSI Driver** - the Container Storage Interface implementation that provisions and mounts volumes
3. **Ceph Daemon Layer** - the actual Ceph processes (MON, OSD, MGR, MDS, RGW)

```text
+----------------------------+
|    Kubernetes API Server   |
+----------------------------+
           |
+----------------------------+
|      Rook Operator         |  -- Watches CRDs, manages Ceph cluster lifecycle
+----------------------------+
           |
+------------------+  +------------------+
|  Ceph CSI Driver |  |  Ceph Daemons    |
|  (RBD + CephFS)  |  |  MON/OSD/MGR/MDS |
+------------------+  +------------------+
           |
+----------------------------+
|    Storage (Disks/PVCs)    |
+----------------------------+
```

## Layer 1: The Rook Operator

The Rook operator is a single deployment in the `rook-ceph` namespace that watches Custom Resource Definitions (CRDs) and translates them into Ceph actions.

```bash
# The operator deployment
kubectl -n rook-ceph get deploy rook-ceph-operator

# CRDs managed by Rook
kubectl get crd | grep ceph.rook.io
```

Key CRDs managed by the operator:

| CRD | Purpose |
|-----|---------|
| `CephCluster` | The entire Ceph cluster configuration |
| `CephBlockPool` | Block storage pools |
| `CephFilesystem` | CephFS shared filesystem |
| `CephObjectStore` | S3-compatible object storage |
| `CephObjectStoreUser` | Object store users |

When you apply a `CephCluster` CR, the operator:
1. Deploys MON pods to establish quorum
2. Deploys MGR pods for management functions
3. Discovers and deploys OSD pods for each device
4. Deploys MDS pods if a CephFilesystem is defined
5. Deploys RGW pods if a CephObjectStore is defined

## Layer 2: Ceph CSI Driver

The Ceph CSI driver is a set of pods that implement the Container Storage Interface spec. There are two CSI drivers deployed by Rook: one for RBD (block) storage and one for CephFS (shared filesystem) storage.

```bash
# CSI provisioner and node plugin pods
kubectl -n rook-ceph get pods | grep csi
```

The CSI layer handles:
- **Dynamic provisioning:** Creates Ceph block images or CephFS subvolumes when a PVC is created
- **Volume attachment:** Attaches the volume to the node where the pod is scheduled
- **Volume mounting:** Mounts the volume into the pod's filesystem
- **Snapshot and resize operations**

```yaml
# A StorageClass that uses the RBD CSI provisioner
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com  # CSI provisioner name
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
```

## Layer 3: Ceph Daemons

The Ceph daemon layer is the actual Ceph storage system running as Kubernetes pods:

**MON (Monitor):** Maintains the cluster map (OSD map, CRUSH map, PG map). Typically 3 or 5 MONs for quorum.

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mon
```

**OSD (Object Storage Daemon):** Stores and retrieves data. Each OSD manages one disk. The minimum recommended count is 3 (one per node for replication).

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

**MGR (Manager):** Provides monitoring, metrics, and dashboard functions. Runs the Ceph Dashboard and Prometheus exporter.

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mgr
```

**MDS (Metadata Server):** Required for CephFS. Manages filesystem metadata.

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds
```

**RGW (RADOS Gateway):** S3/Swift-compatible object storage API. Deployed when a `CephObjectStore` CRD is created.

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-rgw
```

## How a PVC Request Flows Through the Architecture

```text
1. Pod requests a PVC (PersistentVolumeClaim)
2. Kubernetes PVC controller calls the CSI provisioner
3. CSI provisioner creates an RBD image in the Ceph block pool
4. CSI node plugin attaches the image to the target node (via rbd kernel module)
5. CSI mounts the image at the pod's volume mount path
6. Pod reads/writes data through the mounted volume
7. Ceph OSD daemons store the data with replication
```

## Rook Operator Reconciliation Loop

The operator continuously reconciles the desired state (CRDs) with the actual state (running pods):

```bash
# Watch operator logs to see reconciliation events
kubectl -n rook-ceph logs deploy/rook-ceph-operator -f | grep -i reconcil
```

If an OSD pod crashes, the operator detects the discrepancy and restarts it. If a MON loses quorum, the operator automatically removes the failed MON and starts a replacement.

## Summary

Rook's architecture cleanly separates concerns across three layers: the operator handles lifecycle management, the CSI drivers handle Kubernetes-native volume provisioning and mounting, and the Ceph daemons handle actual data storage. Understanding this separation helps you diagnose issues at the right layer - operator logs for cluster state issues, CSI logs for provisioning failures, and Ceph daemon logs for storage-level problems.
