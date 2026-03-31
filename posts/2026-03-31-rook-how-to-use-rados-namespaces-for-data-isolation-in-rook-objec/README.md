# How to Use RADOS Namespaces for Data Isolation in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Rados, Namespaces, Multi-Tenancy, Kubernetes

Description: Use RADOS namespaces in Rook to achieve data isolation between tenants sharing the same Ceph pool without the overhead of separate pools.

---

## Overview

RADOS namespaces provide a lightweight way to isolate objects within a single Ceph pool. Instead of creating separate pools for each tenant or workload, you can assign different namespaces to different users or applications. Rook supports RADOS namespaces for both RBD block storage and CephFS subvolumes, enabling multi-tenancy without the management overhead of many pools.

## RADOS Namespaces in RBD Block Storage

In the StorageClass, set the `radosNamespace` parameter to assign all provisioned images to a specific namespace:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-tenant-a
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  radosNamespace: tenant-a
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Create a separate StorageClass for another tenant:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-tenant-b
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  radosNamespace: tenant-b
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## List Images in a Namespace

Verify that images are correctly namespaced:

```bash
# List all images in tenant-a namespace
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls --namespace tenant-a replicapool

# List all images in tenant-b namespace
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd ls --namespace tenant-b replicapool
```

## Namespace-Scoped Access Keys

To enforce isolation at the access level, create separate Ceph auth keys per namespace:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.tenant-a \
  mon 'profile rbd' \
  osd 'profile rbd pool=replicapool namespace=tenant-a'
```

This key can only access images within the `tenant-a` namespace of the `replicapool`.

## RADOS Namespaces in Object Store

For object storage multi-tenancy, RGW uses its own user-based tenancy model, but you can configure per-user namespace prefixes:

```bash
# Create a user with tenant isolation
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --tenant=tenant-a \
  --uid=user1 \
  --display-name="Tenant A User 1" \
  --access-key=AKIAIOSFODNN7EXAMPLE \
  --secret-key=wJalrXUtnFEMI/K7MDENG
```

Tenant-aware bucket access:

```bash
aws s3 ls s3://tenant-a%3Amy-bucket \
  --endpoint-url http://<rgw-endpoint>
```

## Namespace Quotas

Apply capacity limits to a namespace using pool namespace quotas:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd namespace create replicapool/tenant-a

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_bytes 107374182400
```

## Summary

RADOS namespaces in Rook provide data isolation within a shared pool by namespacing RBD images per tenant or application. Create separate StorageClasses with `radosNamespace` parameters for each tenant, and optionally create namespace-scoped auth keys to enforce access control. This approach enables multi-tenancy without the operational burden of managing dozens of separate pools.
