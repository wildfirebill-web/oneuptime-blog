# How to Remove a CephFS Filesystem

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to safely remove a CephFS filesystem, stop MDS daemons, and delete associated pools without losing other data in your Rook cluster.

---

## Overview

Removing a CephFS filesystem is a destructive operation. Once the filesystem is removed, all files and directories stored in it are permanently deleted. Take a full backup before proceeding. In Rook deployments, deletion is governed by the `preserveFilesystemOnDelete` setting in the `CephFilesystem` CRD.

## Step 1: Verify No Clients Are Connected

Before removing a filesystem, ensure no clients are actively using it:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
ceph fs status myfs
```

If the output shows connected clients, unmount them first:

```text
myfs - 0 clients   # Safe to proceed
myfs - 3 clients   # Must unmount clients first
```

## Step 2: Mark the Filesystem Down

Marking the filesystem `down` prevents new mounts and terminates active MDS sessions:

```bash
ceph fs set myfs down true
```

Verify:

```bash
ceph fs status myfs
```

## Step 3: Remove the Filesystem

Use `ceph fs rm` with the `--yes-i-really-mean-it` safety flag:

```bash
ceph fs rm myfs --yes-i-really-mean-it
```

This removes the CephFS filesystem entry but does not delete the underlying pools.

## Step 4: Remove the Pools (Optional but Recommended)

After removing the filesystem, the metadata and data pools still exist. Delete them if no longer needed:

```bash
# Delete data pool
ceph osd pool delete myfs-data myfs-data --yes-i-really-really-mean-it

# Delete metadata pool
ceph osd pool delete myfs-metadata myfs-metadata --yes-i-really-really-mean-it
```

Note: Pool deletion requires the `mon_allow_pool_delete` configuration to be true:

```bash
ceph config set mon mon_allow_pool_delete true
```

Remember to set it back to false after:

```bash
ceph config set mon mon_allow_pool_delete false
```

## Removing in Rook via CephFilesystem CRD

In Rook, set the `preserveFilesystemOnDelete` field to `false` before deleting the CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  preserveFilesystemOnDelete: false
```

Apply:

```bash
kubectl apply -f cephfilesystem.yaml
```

Then delete the CRD:

```bash
kubectl delete cephfilesystem myfs -n rook-ceph
```

Rook will call `ceph fs rm` and delete the associated pools automatically when `preserveFilesystemOnDelete: false`.

## If preserveFilesystemOnDelete Is True

With `preserveFilesystemOnDelete: true` (the default), deleting the `CephFilesystem` resource stops the MDS daemons but leaves the Ceph filesystem and pools intact. This is the safe default for production.

## Verifying Removal

Confirm the filesystem is gone:

```bash
ceph fs ls
```

Expected: empty output or only other filesystems listed.

Also check that MDS pods were terminated:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds | grep myfs
```

Expected: no pods matching `myfs`.

## Summary

To remove a CephFS filesystem: verify no clients are connected, mark it `down`, run `ceph fs rm <name> --yes-i-really-mean-it`, and optionally delete the metadata and data pools. In Rook, set `preserveFilesystemOnDelete: false` in the `CephFilesystem` CRD before deletion to have Rook handle the full cleanup. Always back up data before proceeding.
