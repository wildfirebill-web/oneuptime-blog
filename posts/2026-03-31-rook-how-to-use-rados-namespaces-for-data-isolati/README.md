# How to Use RADOS Namespaces for Data Isolation in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Rados, Namespaces, Isolation, Kubernetes

Description: Learn how to use RADOS namespaces in Rook to isolate tenant data within a shared Ceph pool while maintaining logical separation without separate pools.

---

## Overview

RADOS namespaces provide a lightweight mechanism to logically isolate data within a single Ceph pool. Rather than creating separate pools per tenant, you can use namespaces to segment objects, enabling per-namespace quotas and access controls without the overhead of pool proliferation.

This is especially useful in the Rook RBD CSI driver, where namespaces allow multiple StorageClasses to share a single pool while keeping their data isolated.

## RADOS Namespaces in RBD StorageClasses

Configure separate StorageClasses for different tenants, each using a different namespace within the same pool:

```yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-tenant-a
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  dataBlockPool: ""
  imageFormat: "2"
  imageFeatures: layering
  radosNamespace: tenant-a
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-tenant-b
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  radosNamespace: tenant-b
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Creating a RADOS Namespace Manually

You can also create a namespace directly via the Ceph CLI:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd namespace create replicapool/tenant-a

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd namespace create replicapool/tenant-b
```

List existing namespaces:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd namespace list replicapool
```

Output:

```text
NAME
tenant-a
tenant-b
```

## Listing Images in a Namespace

View RBD images within a specific namespace:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls replicapool --namespace tenant-a
```

## Setting Namespace Quotas

Apply quotas at the pool level to limit total usage across all namespaces:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_bytes 107374182400
```

For per-namespace quotas, use the Ceph client capability system to restrict a user to a specific namespace:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth add client.tenant-a \
    mon 'profile rbd' \
    osd 'profile rbd pool=replicapool namespace=tenant-a'
```

## RADOS Namespaces in the Object Store

For RGW, RADOS namespaces can be used to partition object store buckets at the storage layer. When creating a zone, you can specify a different namespace prefix for metadata and data:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: zone-primary
  namespace: rook-ceph
spec:
  zoneGroup: my-zonegroup
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
```

## Removing a Namespace

Before removing a namespace, ensure all images within it are deleted:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd namespace remove replicapool/tenant-a
```

## Summary

RADOS namespaces in Rook provide logical data isolation within a shared pool, allowing multiple tenants or applications to coexist without the resource overhead of separate pools. In RBD CSI configurations, the `radosNamespace` parameter in a StorageClass directs volumes to a specific namespace, enabling data separation while sharing pool-level resources like OSDs and placement groups.
