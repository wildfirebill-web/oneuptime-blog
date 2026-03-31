# How to Set preserveFilesystemOnDelete in Rook CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cephfs, Kubernetes, Storage

Description: Learn how to configure preserveFilesystemOnDelete in Rook to prevent accidental data loss when a CephFilesystem CRD is deleted from Kubernetes.

---

## Overview

By default, when you delete a CephFilesystem custom resource from Kubernetes, Rook will also delete the underlying Ceph filesystem and all its data. The `preserveFilesystemOnDelete` field prevents this behavior, allowing you to delete the CRD without destroying the actual filesystem data in Ceph.

This setting is critical for production environments where accidental CRD deletion should not result in data loss.

## What preserveFilesystemOnDelete Does

When set to `true`:
- Deleting the CephFilesystem CRD stops Rook from managing the filesystem
- The Ceph filesystem and its data pools remain intact
- MDS daemons are stopped but the filesystem data is preserved

When set to `false` (default):
- Deleting the CephFilesystem CRD removes the filesystem from Ceph
- All data stored in the filesystem is permanently deleted

## Configuring preserveFilesystemOnDelete

Add the field to your CephFilesystem manifest:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Apply the manifest:

```bash
kubectl apply -f cephfilesystem.yaml
```

## Updating an Existing CephFilesystem

If you have an existing CephFilesystem and want to enable preservation, edit the resource in place:

```bash
kubectl -n rook-ceph edit cephfilesystem myfs
```

Change the field value:

```yaml
spec:
  preserveFilesystemOnDelete: true
```

Alternatively, patch it directly:

```bash
kubectl -n rook-ceph patch cephfilesystem myfs \
  --type merge \
  -p '{"spec":{"preserveFilesystemOnDelete":true}}'
```

## Verifying the Setting

Confirm the setting was applied:

```bash
kubectl -n rook-ceph get cephfilesystem myfs -o jsonpath='{.spec.preserveFilesystemOnDelete}'
```

Expected output:

```text
true
```

## Testing the Behavior

To safely test the preservation behavior in a non-production environment:

1. Create a test filesystem with preserveFilesystemOnDelete enabled
2. Write some test data to the filesystem
3. Delete the CephFilesystem CRD
4. Verify the filesystem still exists in Ceph

```bash
# Step 1: Delete the CRD
kubectl -n rook-ceph delete cephfilesystem myfs

# Step 2: Check filesystem still exists in Ceph
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs ls
```

Expected output showing the filesystem persists:

```text
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-replicated ]
```

## Re-adopting a Preserved Filesystem

After deleting the CRD, you can re-adopt the filesystem by reapplying the same manifest:

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook will detect the existing filesystem and resume managing it without recreating or destroying it.

## Summary

The `preserveFilesystemOnDelete` field in CephFilesystem CRDs is a safety mechanism that prevents data loss when the Kubernetes resource is accidentally or intentionally deleted. Setting it to `true` ensures the underlying Ceph filesystem data remains intact. This is a recommended setting for all production CephFilesystem deployments, especially when the filesystem holds critical application data.
