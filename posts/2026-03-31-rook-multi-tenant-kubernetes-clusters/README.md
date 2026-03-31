# How to Configure Rook-Ceph for Multi-Tenant Kubernetes Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multi-Tenant, Kubernetes, Namespace, Storage

Description: Learn how to configure Rook-Ceph storage isolation for multi-tenant Kubernetes clusters using separate pools, quotas, StorageClasses, and namespace-scoped access controls.

---

Multi-tenant Kubernetes clusters require storage isolation between tenants to ensure data privacy, fair resource allocation, and prevent one tenant from impacting another. Rook-Ceph supports multi-tenancy through pool-level isolation, quotas, and RBAC-scoped StorageClasses.

## Isolation Strategies

Rook-Ceph supports several levels of tenant isolation:

1. **Pool-per-tenant** - Each tenant gets their own Ceph pool with separate quotas
2. **Namespace-scoped StorageClass** - Limit which namespaces can provision from which pools
3. **ObjectStore per tenant** - Dedicated S3 endpoints for stronger object storage isolation
4. **CephFS subvolume groups** - Namespace-level isolation within a shared filesystem

## Creating Per-Tenant Pools

Create separate block pools for each tenant:

```bash
# Tenant A pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool create tenant-a-rbd 32 32

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota tenant-a-rbd max_bytes 107374182400

# Tenant B pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool create tenant-b-rbd 32 32

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota tenant-b-rbd max_bytes 107374182400
```

## Creating Tenant-Specific StorageClasses

Define a StorageClass per tenant pointing to their dedicated pool:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tenant-a-storage
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: tenant-a-rbd
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

## Restricting StorageClass Access with RBAC

Prevent tenant-b from using tenant-a's StorageClass:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tenant-a-storage-user
rules:
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    resourceNames: ["tenant-a-storage"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-storage-binding
  namespace: tenant-a
subjects:
  - kind: Group
    name: tenant-a-users
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: tenant-a-storage-user
  apiGroup: rbac.authorization.k8s.io
```

## Multi-Tenant CephFS with Subvolume Groups

For shared filesystem isolation, use CephFS subvolume groups:

```bash
# Create subvolume groups per tenant
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs subvolumegroup create myfs tenant-a

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs subvolumegroup create myfs tenant-b

# Create subvolumes with size limits
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs subvolume create myfs data tenant-a --size 107374182400
```

## Setting Resource Quotas

Combine Ceph quotas with Kubernetes ResourceQuotas:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: tenant-a
spec:
  hard:
    requests.storage: "100Gi"
    persistentvolumeclaims: "10"
    tenant-a-storage.storageclass.storage.k8s.io/requests.storage: "100Gi"
```

## Summary

Multi-tenant Rook-Ceph deployments use a combination of per-tenant pools with quotas, dedicated StorageClasses per tenant, Kubernetes RBAC to restrict StorageClass usage, and CephFS subvolume groups for filesystem isolation. ResourceQuotas at the namespace level enforce limits that work alongside Ceph pool quotas to provide comprehensive tenant resource governance.
