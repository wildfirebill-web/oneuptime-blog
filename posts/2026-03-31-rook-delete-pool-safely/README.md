# How to Delete Pools Safely in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool, Kubernetes, Storage

Description: Learn how to safely delete Ceph pools in Rook-managed clusters, avoiding data loss and respecting pool deletion protection flags.

---

Deleting a Ceph pool is an irreversible operation that removes all data stored in that pool. Ceph has built-in safeguards to prevent accidental deletion, and Rook adds its own layer of protection. Understanding these mechanisms is essential before proceeding.

## Pool Deletion Safeguards

By default, Ceph prevents pool deletion unless two specific configuration options are enabled. This is intentional - a pool delete is catastrophic and cannot be undone.

```bash
# These must be set before deletion is permitted
ceph config set mon mon_allow_pool_delete true
ceph osd pool set <pool-name> nodelete false
```

Rook also has its own protection at the CRD level. The `spec.preservePoolsOnDelete` field on `CephBlockPool` and `CephFilesystem` objects controls whether Rook deletes the underlying Ceph pool when the Kubernetes resource is deleted.

## Check Pool Flags Before Deleting

Before deleting any pool, inspect its flags to understand what protections are active:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get <pool-name> all
```

Look for the `nodelete` flag. If it is set, you must unset it before deletion:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set <pool-name> nodelete false
```

## Enable Pool Deletion at Monitor Level

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mon mon_allow_pool_delete true
```

This flag enables pool deletion cluster-wide. Re-disable it after your deletion task is complete to restore protection:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mon mon_allow_pool_delete false
```

## Delete a Pool via Rook CephBlockPool

The cleanest approach is to delete the Kubernetes resource and let Rook handle the pool:

```yaml
# First, ensure preservePoolsOnDelete is false in the spec
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: mypool
  namespace: rook-ceph
spec:
  preservePoolsOnDelete: false
  replicated:
    size: 3
```

Then delete the resource:

```bash
kubectl delete cephblockpool mypool -n rook-ceph
```

## Delete a Pool via CLI

For pools not managed by a Rook CRD, use the Ceph CLI directly:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool delete <pool-name> <pool-name> --yes-i-really-really-mean-it
```

Note the pool name must be repeated twice as a confirmation mechanism. The `--yes-i-really-really-mean-it` flag is mandatory.

## Verify Pool Removal

After deletion, confirm the pool no longer exists:

```bash
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool ls
```

Also check that any StorageClasses or PVCs referencing the deleted pool are cleaned up to avoid orphaned Kubernetes objects.

## Pre-Deletion Checklist

Before deleting any pool:

- Confirm no active workloads are writing to the pool
- Delete or migrate any PVCs backed by the pool
- Remove associated StorageClass resources
- Back up any critical data using RADOS export or volume snapshots
- Verify no RBD mirrors are replicating from this pool

## Summary

Safely deleting a Ceph pool in Rook requires disabling the `nodelete` flag, enabling `mon_allow_pool_delete`, and either removing the Rook CRD or using the CLI double-confirmation syntax. Always re-enable deletion protection after completing the operation, and clean up associated Kubernetes StorageClass and PVC resources.
