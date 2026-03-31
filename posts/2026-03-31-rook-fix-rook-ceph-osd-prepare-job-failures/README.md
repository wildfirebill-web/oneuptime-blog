# How to Fix Rook-Ceph OSD Prepare Job Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, OSD, Kubernetes

Description: Learn how to diagnose and fix Rook-Ceph OSD prepare job failures by inspecting job logs, checking device availability, and resolving partition, permission, and node selector issues.

---

## Understanding OSD Prepare Jobs

When Rook provisions a new OSD, it creates a Kubernetes Job (named `rook-ceph-osd-prepare-<node>`) that prepares the disk and initializes BlueStore. If this job fails, the OSD is never created and cluster capacity is not expanded.

## Check OSD Prepare Job Status

```bash
# List all OSD prepare jobs
kubectl -n rook-ceph get jobs -l app=rook-ceph-osd-prepare

# Check job status
kubectl -n rook-ceph describe job rook-ceph-osd-prepare-worker-node-1

# View job pod logs
kubectl -n rook-ceph logs job/rook-ceph-osd-prepare-worker-node-1
```

## Common Cause 1: Device Already Has Partitions

Rook requires raw, empty block devices. If the device has existing partitions or a filesystem, the prepare job fails:

```bash
# On the target node, check device state
lsblk /dev/sdb
fdisk -l /dev/sdb
```

If the device has existing data, wipe it completely:

```bash
# DANGER: This destroys all data on the device
wipefs -a /dev/sdb
dd if=/dev/zero of=/dev/sdb bs=4096 count=100
```

## Common Cause 2: Device Not Listed in CephCluster Spec

Verify the device name in your CephCluster matches the actual device on the node:

```bash
# Check devices on the node
kubectl debug node/worker-node-1 -it --image=ubuntu -- lsblk

# Verify CephCluster spec
kubectl -n rook-ceph get cephcluster rook-ceph -o yaml | grep -A20 "storage:"
```

The spec should reference the correct device:

```yaml
storage:
  nodes:
    - name: "worker-node-1"
      devices:
        - name: "sdb"
```

## Common Cause 3: Node Selector Mismatch

The OSD prepare job uses node affinity. If the node label is missing, the pod will not be scheduled:

```bash
# Check node labels
kubectl get node worker-node-1 --show-labels

# Required label for Rook
kubectl label node worker-node-1 ceph-osd-node=enabled
```

## Common Cause 4: Insufficient Permissions (SELinux/AppArmor)

On nodes with strict security policies, the OSD prepare job may fail due to permission errors:

```bash
# Check for permission errors in the job logs
kubectl -n rook-ceph logs job/rook-ceph-osd-prepare-worker-node-1 | grep -i "permission\|denied\|error"

# On the node, check for SELinux denials
ausearch -m avc -ts recent | grep ceph
```

Rook requires privileged access to prepare block devices. Verify the operator has the required security context:

```bash
kubectl -n rook-ceph get deploy rook-ceph-operator -o yaml | grep -A5 "securityContext"
```

## Common Cause 5: Device Busy or Mounted

If the device is currently in use or mounted, Rook cannot prepare it:

```bash
# On the node, check if device is mounted or open
mount | grep sdb
lsof /dev/sdb
fuser /dev/sdb
```

Unmount and close any processes using the device before letting Rook proceed.

## Force Retry the OSD Prepare Job

After fixing the root cause, delete the failed job to trigger a retry:

```bash
kubectl -n rook-ceph delete job rook-ceph-osd-prepare-worker-node-1
```

The Rook operator will automatically recreate the job.

## Summary

Rook-Ceph OSD prepare job failures are most commonly caused by devices with existing partitions, incorrect device names in the CephCluster spec, missing node labels causing scheduling failures, security policy blocks, or devices that are currently mounted. After identifying the cause through job logs and node-level inspection, wiping the device, correcting the spec, or fixing permissions resolves the issue. Delete the failed job to trigger an automatic retry by the Rook operator.
