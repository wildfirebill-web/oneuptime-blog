# How to Fix 'no osds' Error When Deploying Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, OSD, Deployment, Error

Description: Diagnose and fix the 'no osds' error when deploying Rook-Ceph by checking disk requirements, operator logs, and cluster configuration.

---

## Introduction

The "no osds" error is one of the most common issues when first deploying Rook-Ceph. It means the operator found no eligible disks or partitions to provision OSDs on. This guide covers the most frequent causes and their fixes.

## Symptoms

```text
ceph status shows:
  cluster:
    health: HEALTH_ERR
    ...
  osd: 0 osds: 0 up, 0 in
```

Or in the operator logs:

```text
failed to configure OSDs for node "worker-1": no devices were configured
```

## Cause 1 - Disks Are Not Raw (Most Common)

Rook requires disks to be completely raw - no filesystem, no partition table.

Check disk state on the node:

```bash
lsblk -f /dev/sdb
```

If you see a filesystem or partition, wipe it:

```bash
wipefs -a /dev/sdb
dd if=/dev/zero of=/dev/sdb bs=4096 count=100
```

## Cause 2 - Incorrect useAllDevices or deviceFilter

Check your CephCluster spec:

```yaml
storage:
  useAllDevices: true
  # Or explicitly list devices:
  nodes:
  - name: "worker-1"
    devices:
    - name: "sdb"
```

If `useAllDevices: false` and no `nodes` or `devices` are specified, no OSDs will be created.

Verify the operator sees the devices:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-operator | grep -i "osd\|device\|disk"
```

## Cause 3 - Node Labels Not Matching

Rook uses node labels for OSD scheduling. Verify nodes are labeled correctly:

```bash
kubectl get nodes --show-labels | grep storage
```

If missing, add the label:

```bash
kubectl label nodes worker-1 role=storage-node
```

And update the CephCluster spec:

```yaml
placement:
  osd:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: role
            operator: In
            values:
            - storage-node
```

## Cause 4 - LVM or dm-crypt Partitions

Disks with LVM physical volumes or dm-crypt mappings are not eligible:

```bash
# Check for LVM
pvs /dev/sdb

# Remove LVM if present
pvremove /dev/sdb
```

## Cause 5 - Operator Cannot Access Nodes

Check if the operator pod is running:

```bash
kubectl -n rook-ceph get pods | grep operator
kubectl -n rook-ceph logs deploy/rook-ceph-operator | tail -50
```

## Verifying the Fix

After fixing disks or configuration, restart the operator to trigger re-detection:

```bash
kubectl -n rook-ceph rollout restart deploy/rook-ceph-operator
```

Then watch OSD pods come up:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd --watch
```

## Summary

The "no osds" error typically results from disks that are not completely raw, incorrect `useAllDevices` or device filter configuration, or missing node labels. Wiping disks thoroughly with `wipefs` and `dd`, then verifying the CephCluster spec matches the actual disk layout, resolves the issue in most cases.
