# How to Use Global CephFS Settings with ceph fs flag set

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Configuration, Administration

Description: Learn how to configure global CephFS flags using ceph fs flag set to manage cluster-wide filesystem behavior in Rook-Ceph deployments.

---

## Overview

The `ceph fs flag set` command manages global flags that apply to the entire CephFS subsystem across all filesystems in a Ceph cluster, rather than settings on a specific filesystem. These flags are stored in the MDS map and affect cluster-wide behavior. The most commonly used global flag is `enable_multiple`, which unlocks support for multiple CephFS filesystems.

## List Current Global Flags

To see all currently set global CephFS flags:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs dump | grep flags
```

## Enable Multiple Filesystems

The primary use of `ceph fs flag set` is enabling multi-filesystem support:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs flag set enable_multiple true --yes-i-really-mean-it
```

The `--yes-i-really-mean-it` flag is required as a safety confirmation because this is a cluster-wide change.

## Disable Multiple Filesystem Support

If you want to revert to single-filesystem mode (only safe if you have only one filesystem):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs flag set enable_multiple false
```

## Set vs Per-Filesystem Configuration

It is important to distinguish between global flags (set with `ceph fs flag set`) and per-filesystem settings (set with `ceph fs set`):

```bash
# Global flag - affects all filesystems
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs flag set enable_multiple true --yes-i-really-mean-it

# Per-filesystem setting - affects only the named filesystem
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set cephfs max_file_size 1099511627776
```

## Verify the Flag Was Set

After setting a flag, confirm it is active in the MDS map:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs dump
```

Look for the `flags` field near the top of the output. A value of `1` or the flag name indicates it is set.

## Practical Workflow: Enabling Multiple Filesystems

Here is a complete workflow for enabling multiple filesystem support and creating a second filesystem:

```bash
# Step 1: Enable the global flag
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs flag set enable_multiple true --yes-i-really-mean-it

# Step 2: Verify the flag
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs dump | head -20

# Step 3: Create a second filesystem via CRD
kubectl apply -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs2
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
EOF

# Step 4: Confirm both filesystems exist
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs ls
```

## Summary

The `ceph fs flag set` command controls cluster-wide CephFS behavior, with `enable_multiple` being the most commonly used flag. Unlike per-filesystem `ceph fs set` commands, `fs flag set` modifies the global MDS map. Understanding the distinction between global flags and per-filesystem settings is essential for correct CephFS administration in Rook-Ceph environments.
