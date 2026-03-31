# How to Implement Data Classification with Ceph Pools

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Data Classification, Storage Pool, Security, Compliance, Namespace

Description: Implement data classification in Ceph by mapping data sensitivity levels to separate pools, StorageClasses, and encryption configurations for regulated workloads.

---

## What is Data Classification?

Data classification assigns data to categories based on sensitivity (e.g., Public, Internal, Confidential, Restricted) and applies storage controls appropriate for each level. In Kubernetes with Ceph, data classification maps to separate pools with different replication policies, encryption configurations, and access controls.

## Classification Tiers and Their Storage Requirements

| Classification | Replication | Encryption | Access | Audit |
|---------------|-------------|------------|--------|-------|
| Public | 2 copies | None | Open | None |
| Internal | 3 copies | Optional | Authenticated | Basic |
| Confidential | 3 copies | Required | RBAC | Full |
| Restricted | 3 copies + offsite | Required + KMS | Break-glass | Immutable |

## Creating Classification-Specific Pools

```yaml
# Public data pool - reduced replication for cost efficiency
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: public-data-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
  parameters:
    compression_mode: passive
---
# Confidential data pool - full replication with compression
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: confidential-data-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    compression_mode: none  # Encryption defeats compression
---
# Restricted data pool - replicated 3 across isolated failure domains
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: restricted-data-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  failureDomain: rack  # Higher failure domain for added resilience
```

## StorageClasses per Classification Level

```yaml
# Internal StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-internal
  annotations:
    classification: internal
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: internal-data-pool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
---
# Confidential StorageClass with encryption
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-confidential
  annotations:
    classification: confidential
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: confidential-data-pool
  encrypted: "true"
  encryptionKMSID: vault-confidential
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain  # Retain for classified data
```

## Enforcing Classification with Kubernetes RBAC

Use Kubernetes RBAC to restrict which namespaces can use which StorageClasses:

```yaml
# Allow only the security namespace to provision confidential PVCs
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: confidential-storage-user
rules:
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    resourceNames: ["rook-ceph-confidential"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: confidential-storage-binding
  namespace: secure-workloads
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: confidential-storage-user
subjects:
  - kind: ServiceAccount
    name: secure-app-sa
    namespace: secure-workloads
```

## Using OPA/Gatekeeper to Enforce Classification

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: StorageClassRequired
metadata:
  name: require-classified-storage
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["PersistentVolumeClaim"]
    namespaces: ["secure-workloads", "finance"]
  parameters:
    allowedClasses: ["rook-ceph-confidential", "rook-ceph-restricted"]
```

## Summary

Data classification with Ceph maps sensitivity levels to distinct pools with appropriate replication, encryption, and reclaim policies. Kubernetes StorageClasses expose these pools while RBAC and policy engines (OPA/Gatekeeper) enforce which workloads can access which classification level. The combination of infrastructure-level pool separation and Kubernetes-level access controls provides defense in depth for multi-tenancy environments with mixed data sensitivity requirements.
