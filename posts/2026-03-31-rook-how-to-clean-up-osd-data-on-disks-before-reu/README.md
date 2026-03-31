# How to Clean Up OSD Data on Disks Before Reuse in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, OSD, Disk Cleanup, Kubernetes, Storage

Description: Learn how to properly clean up OSD data from disks before reusing them in a new Rook-Ceph deployment or for other purposes.

---

## Overview

When an OSD disk is removed from a Rook-Ceph cluster or a cluster is deleted, the disk retains partition tables, LVM data, and Ceph metadata. Before reusing these disks, you must clean them completely. Rook's automatic disk detection will skip disks it considers "in use" based on this leftover data.

## Why Cleanup is Required

Ceph writes:
- BlueStore metadata to the beginning of the disk
- LVM logical volumes and volume groups (for some configurations)
- Partition tables for OSD data and WAL/DB devices

If you try to add a "dirty" disk to a new Rook cluster, the operator will not use it and the OSD will fail to provision.

## Step 1 - Remove the OSD from the Cluster First

If the disk is still active in a running cluster, gracefully remove the OSD before cleaning the disk:

```bash
# Get the OSD ID for the disk
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd tree

# Mark the OSD out
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd out osd.2

# Wait for data migration to complete
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph -w

# Remove the OSD
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd purge osd.2 --yes-i-really-mean-it
```

## Step 2 - Delete the OSD Pod and PVC

Remove the OSD deployment from Kubernetes:

```bash
kubectl -n rook-ceph delete deploy rook-ceph-osd-2
```

## Step 3 - Identify the Disk to Clean

On the storage node, identify the OSD disk:

```bash
lsblk -f
```

Find disks with `ceph` labels:

```bash
lsblk -o NAME,TYPE,FSTYPE,LABEL,MOUNTPOINT | grep ceph
```

List LVM volumes related to Ceph:

```bash
lvdisplay | grep ceph
vgdisplay | grep ceph
pvdisplay
```

## Step 4 - Remove LVM Data

Remove Ceph LVM volumes before wiping the disk:

```bash
# List Ceph volume groups
VGS=$(vgs --noheadings -o vg_name | grep ceph)

# Remove logical volumes in each VG
for vg in $VGS; do
  lvremove -f /dev/$vg/*
  vgremove -f $vg
done

# Remove physical volumes
pvremove /dev/sdX
```

## Step 5 - Wipe the Disk

After removing LVM data, wipe the disk completely:

```bash
DISK=/dev/sdX

# Zap all partitions and GPT/MBR data
sgdisk --zap-all $DISK

# Clear any remaining filesystem signatures
wipefs --all $DISK

# Overwrite the beginning of the disk (BlueStore metadata)
dd if=/dev/zero of=$DISK bs=1M count=100 oflag=direct,dsync

# If the disk supports it, send TRIM/discard
blkdiscard $DISK

# Re-read partition table
partprobe $DISK
```

## Step 6 - Verify the Disk is Clean

Confirm no data signatures remain:

```bash
blkid $DISK
lsblk -f $DISK
```

Both commands should return empty output or show an unformatted disk.

## Step 7 - Using the Rook Disk Cleanup Job

Rook provides an optional disk cleanup job that can run on nodes automatically. Apply the job template:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: disk-clean
  namespace: rook-ceph
spec:
  template:
    spec:
      containers:
        - name: disk-clean
          image: quay.io/ceph/ceph:v18
          securityContext:
            privileged: true
          command: ["/bin/bash", "-c"]
          args:
            - |
              DISK=/dev/sdX
              sgdisk --zap-all $DISK
              dd if=/dev/zero of=$DISK bs=1M count=100 oflag=direct,dsync
              blkdiscard $DISK
          volumeMounts:
            - name: devices
              mountPath: /dev
      volumes:
        - name: devices
          hostPath:
            path: /dev
      nodeName: storage-node-1
      restartPolicy: Never
```

## Summary

Cleaning OSD disks before reuse in Rook requires removing the OSD from the Ceph cluster, deleting LVM volumes and volume groups associated with Ceph, wiping partition tables using `sgdisk`, overwriting metadata with `dd`, and issuing a `blkdiscard` for SSDs. This ensures Rook's disk detection does not skip the device and that no leftover Ceph data interferes with new deployments.
