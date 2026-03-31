# How to Fix RBD PVC Not Binding in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, RBD, PVC, Troubleshooting

Description: Diagnose and fix RBD PersistentVolumeClaims stuck in Pending state in Rook-Ceph by identifying provisioner, pool, and secret configuration issues.

---

## Overview

An RBD PVC that does not bind remains in `Pending` state indefinitely. The issue could lie with the CSI provisioner, the underlying Ceph pool, StorageClass configuration, or missing secrets. This guide provides a step-by-step diagnostic approach.

## Step 1: Check PVC Events

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Focus on the `Events` section at the bottom. Common error patterns:

```text
Warning  ProvisioningFailed  Failed to provision volume: ...
Warning  ProvisioningFailed  rpc error: code = Internal
Warning  ProvisioningFailed  error getting secret
```

## Step 2: Check Provisioner Logs

```bash
kubectl logs -n rook-ceph \
  -l app=csi-rbdplugin-provisioner \
  -c csi-provisioner \
  --tail=100 | grep -E "error|Error|WARN|failed"
```

Also check the RBD plugin sidecar:

```bash
kubectl logs -n rook-ceph \
  -l app=csi-rbdplugin-provisioner \
  -c csi-rbdplugin \
  --tail=100 | grep -E "error|Error"
```

## Step 3: Verify Required Secrets

RBD provisioning requires two secrets. Confirm they exist:

```bash
kubectl get secret -n rook-ceph \
  rook-csi-rbd-provisioner \
  rook-csi-rbd-node
```

If missing, they may have been accidentally deleted. Recreate from the Rook example:

```bash
kubectl get secret -n rook-ceph rook-ceph-admin-keyring -o yaml
```

## Step 4: Verify the CephBlockPool Exists

```bash
kubectl get cephblockpool -n rook-ceph
```

Ensure the pool referenced in the StorageClass is listed and has `Ready` phase. Create it if missing:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```

## Step 5: Verify Pool Health in Ceph

```bash
kubectl exec -n rook-ceph -it deploy/rook-ceph-tools -- \
  ceph osd pool ls detail
```

A pool that exists in Kubernetes but not in Ceph indicates the operator failed to create it. Check operator logs:

```bash
kubectl logs -n rook-ceph \
  -l app=rook-ceph-operator | grep -i pool
```

## Step 6: Check RBD Image Creation

When provisioning is triggered, Rook creates an RBD image in the pool. Verify whether the image was created:

```bash
kubectl exec -n rook-ceph -it deploy/rook-ceph-tools -- \
  rbd ls replicapool
```

If no image appears after a PVC is created, the provisioner is failing before the RBD create call.

## Step 7: StorageClass Parameter Validation

A complete, correct RBD StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Summary

RBD PVCs stuck in Pending state require checking PVC events, provisioner logs, secret existence, pool health, and StorageClass parameters in sequence. Most failures trace back to a missing secret, a non-existent pool, or a misconfigured StorageClass parameter. Correct the root cause and the PVC will bind automatically once provisioning succeeds.
