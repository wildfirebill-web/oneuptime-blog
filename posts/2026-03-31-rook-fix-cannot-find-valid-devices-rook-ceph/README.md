# How to Fix 'cannot find valid devices' in Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Troubleshooting, OSD, Device Discovery

Description: Learn how to fix the 'cannot find valid devices' error in Rook-Ceph by checking device discovery configuration, device filters, udev rules, and node disk availability.

---

## Understanding the Error

The "cannot find valid devices" error appears in the Rook operator or OSD prepare job logs when Rook scans a node but finds no suitable disks for OSD creation. This can happen even when disks are physically present if they do not meet Rook's eligibility criteria.

Check for this error in the operator logs:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-operator | grep -i "cannot find\|no valid\|no devices"
```

## Cause 1: Device Has Existing Filesystem or Partitions

Rook only claims completely raw devices. Check each disk:

```bash
# On the target node
lsblk -f

# Example output - device with existing filesystem is REJECTED by Rook:
# sdb      ext4   ...   /mnt/data
# sdc                           <- empty, no filesystem = eligible
```

To make a device eligible, wipe it:

```bash
# WARNING: destroys all data
wipefs --all /dev/sdb
sgdisk --zap-all /dev/sdb
```

## Cause 2: Device Filter Excludes the Disk

If your CephCluster spec uses a `deviceFilter`, ensure the regex matches your device names:

```yaml
spec:
  storage:
    useAllDevices: false
    deviceFilter: "^sd[b-f]"  # Only matches sdb through sdf
```

Verify the filter matches your actual devices:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph -o yaml | grep -A10 "storage:"

# Test the regex on your device list
ls /dev/sd* | grep -E "^/dev/sd[b-f]$"
```

## Cause 3: useAllNodes or useAllDevices is False Without Explicit Config

If both `useAllNodes` and `useAllDevices` are false and no node-specific devices are listed, Rook finds nothing:

```yaml
# Problematic config - no devices specified
spec:
  storage:
    useAllNodes: false
    useAllDevices: false
    # nodes: not specified!
```

Add explicit node and device entries:

```yaml
spec:
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: "worker-node-1"
        devices:
          - name: "sdb"
          - name: "sdc"
      - name: "worker-node-2"
        devices:
          - name: "sdb"
```

## Cause 4: Device Claimed by LVM or multipath

Devices managed by LVM volume groups or multipath daemon are not available to Rook:

```bash
# Check LVM
pvs
vgs
lvs

# Check multipath
multipath -ll

# Remove from LVM if needed
vgremove <vg-name>
pvremove /dev/sdb
```

## Cause 5: Kernel Module or udev Issues

Sometimes kernel device nodes exist but are not accessible. Check udev events:

```bash
udevadm info /dev/sdb
udevadm test /sys/class/block/sdb

# Trigger udev rescan
udevadm trigger --type=devices
udevadm settle
```

## Verify After Fixes

After making disks eligible, restart the Rook operator to trigger a new device scan:

```bash
kubectl -n rook-ceph rollout restart deploy/rook-ceph-operator

# Watch for new OSD prepare jobs
kubectl -n rook-ceph get jobs -w
```

## Summary

The "cannot find valid devices" error in Rook-Ceph is resolved by ensuring target disks are completely raw (no filesystems or partitions), verifying that deviceFilter regex patterns match actual device names, explicitly listing nodes and devices when useAllDevices is false, and removing disks from LVM or multipath control. After fixing disk eligibility, restarting the Rook operator triggers a fresh device discovery scan.
