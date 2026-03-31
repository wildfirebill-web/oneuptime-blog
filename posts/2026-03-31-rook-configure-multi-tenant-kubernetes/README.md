# How to Configure Rook-Ceph for Multi-Tenant Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multi-Tenant, Kubernetes, RBAC, Namespace, Security

Description: Configure Rook-Ceph for multi-tenant Kubernetes clusters with namespace isolation, per-tenant storage classes, quotas, and RBAC to prevent cross-tenant data access.

---

## Multi-Tenancy Goals

In a multi-tenant Kubernetes cluster, different teams or customers share the same Rook-Ceph backend but must be isolated from each other. Goals include: separate storage pools per tenant, namespace-scoped access, quotas to prevent one tenant starving others, and audit trails.

## Create Per-Tenant Pools

```yaml
# Tenant A pool
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: tenant-a-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  quotas:
    maxBytes: 10995116277760   # 10 TB quota for tenant A
---
# Tenant B pool
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: tenant-b-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  quotas:
    maxBytes: 5497558138880    # 5 TB quota for tenant B
```

## Create Per-Tenant Storage Classes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-tenant-a
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: tenant-a-pool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
allowedTopologies: []
allowVolumeExpansion: true
```

## Restrict Storage Class Access via RBAC

```yaml
# Allow only tenant-a namespace to use tenant-a storage class
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-tenant-a-storage
rules:
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    resourceNames: ["ceph-tenant-a"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-storage-access
  namespace: tenant-a
subjects:
  - kind: Group
    name: tenant-a-developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: use-tenant-a-storage
  apiGroup: rbac.authorization.k8s.io
```

## Apply Namespace Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: tenant-a
spec:
  hard:
    requests.storage: "2Ti"
    persistentvolumeclaims: "50"
    ceph-tenant-a.storageclass.storage.k8s.io/requests.storage: "2Ti"
```

## CephFS Subvolume Groups for Filesystem Isolation

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystemSubVolumeGroup
metadata:
  name: tenant-a-subvolumegroup
  namespace: rook-ceph
spec:
  filesystemName: cephfs
  quota:
    maxBytes: 1073741824000   # 1 TB
    maxFiles: 1000000
```

## Audit Cross-Tenant Access Attempts

```bash
# Monitor which namespaces are creating PVCs
kubectl get pvc --all-namespaces | grep tenant

# Check Ceph audit logs for unauthorized access
ceph config set global debug_rgw_access 1
kubectl -n rook-ceph logs deploy/rook-ceph-rgw-my-store | grep "denied\|unauthorized"
```

## Summary

Multi-tenant Rook-Ceph isolation requires per-tenant Ceph pools with size quotas, dedicated StorageClasses with namespace-scoped RBAC bindings, and Kubernetes ResourceQuotas to prevent storage exhaustion. CephFS subvolume groups add filesystem-level quotas and isolation for shared filesystem workloads.
