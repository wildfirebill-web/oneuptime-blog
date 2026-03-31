# How to Fix StorageClass Misconfiguration in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, StorageClass, Troubleshooting, CSI

Description: Identify and fix common StorageClass misconfigurations in Rook-Ceph that prevent PVC provisioning and cause persistent volume binding failures.

---

## Overview

A misconfigured StorageClass is a frequent cause of PVC provisioning failures in Rook-Ceph. Issues range from wrong provisioner names and missing secrets to incorrect pool references. This guide covers systematic diagnosis and correction.

## Diagnosing StorageClass Issues

Start by describing the failing PVC:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Look for error messages in the `Events` section such as:

```text
Failed to provision volume: rpc error: code = Internal desc = ...
Failed to provision volume: StorageClass not found
provisioner: rook-ceph.rbd.csi.ceph.com not found
```

Inspect the StorageClass used by the PVC:

```bash
kubectl describe storageclass rook-ceph-block
```

## Common Misconfigurations

### Wrong Provisioner Name

The provisioner name must exactly match the CSI driver name registered in the cluster:

```bash
# Check registered CSI drivers
kubectl get csidrivers
```

Expected provisioner names:
- RBD: `rook-ceph.rbd.csi.ceph.com`
- CephFS: `rook-ceph.cephfs.csi.ceph.com`

Correct StorageClass provisioner:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
```

### Missing or Wrong clusterID

The `clusterID` parameter must match the Rook namespace:

```bash
# Get the correct clusterID
kubectl get cephcluster -n rook-ceph -o jsonpath='{.items[0].metadata.namespace}'
```

In the StorageClass parameters:

```yaml
parameters:
  clusterID: rook-ceph
```

### Missing Secret References

CSI requires secrets for provisioner and node stage operations:

```yaml
parameters:
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
```

Verify the secrets exist:

```bash
kubectl get secret -n rook-ceph | grep rook-csi-rbd
```

### Wrong Pool Name

The `pool` parameter must reference an existing CephBlockPool:

```bash
kubectl get cephblockpool -n rook-ceph
```

Ensure the StorageClass references the correct pool:

```yaml
parameters:
  pool: replicapool
```

## Applying the Fixed StorageClass

StorageClass is immutable after creation for most parameters. Delete and recreate it:

```bash
kubectl delete storageclass rook-ceph-block
kubectl apply -f rook-storageclass-fixed.yaml
```

Existing PVs are not affected - they store the provisioner information at creation time.

## Verifying the Fix

Create a test PVC to confirm the fixed StorageClass works:

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
EOF

kubectl get pvc test-pvc -w
```

## Summary

StorageClass misconfigurations in Rook-Ceph are fixed by ensuring the provisioner name matches the registered CSI driver, clusterID matches the Rook namespace, all required secrets exist, and the pool parameter references a real CephBlockPool. Delete and recreate the StorageClass when parameters need changing.
