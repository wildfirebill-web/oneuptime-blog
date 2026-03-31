# How to Use rbd-fuse for Mounting RBD Images as Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, rbd-fuse, FUSE, Debugging

Description: Use rbd-fuse to expose Ceph RBD images as regular files in a FUSE-mounted directory, enabling image inspection, copying, and manipulation with standard tools.

---

## What is rbd-fuse

`rbd-fuse` mounts a Ceph RBD pool as a FUSE filesystem where each RBD image appears as a file. This allows you to use standard tools like `cp`, `dd`, and `rsync` to interact with RBD images without needing kernel block device drivers.

## Install and Configure

```bash
# Install rbd-fuse
apt-get install -y librbd-dev fuse   # Debian/Ubuntu
yum install -y librbd1 fuse          # RHEL/CentOS

# Load FUSE module
modprobe fuse
```

## Mount an RBD Pool

```bash
# Create mount point
mkdir -p /mnt/rbd-pool

# Mount the pool - each image appears as a file
rbd-fuse \
  -p rbd \
  /mnt/rbd-pool

# List images as files
ls -lh /mnt/rbd-pool/
```

## Get Credentials from Rook

```bash
# Extract config and keyring from Rook
kubectl -n rook-ceph get secret rook-ceph-admin-keyring \
  -o jsonpath='{.data.keyring}' | base64 -d > /tmp/keyring

kubectl -n rook-ceph get cm rook-ceph-config \
  -o jsonpath='{.data.ceph\.conf}' > /tmp/ceph.conf

# Mount with explicit config
rbd-fuse \
  -c /tmp/ceph.conf \
  -k /tmp/keyring \
  -p rbd \
  /mnt/rbd-pool
```

## Copy an RBD Image to a Local File

```bash
# Copy an entire RBD image to a local backup file
cp /mnt/rbd-pool/myimage /tmp/myimage-backup.img

# Or use dd for more control
dd if=/mnt/rbd-pool/myimage \
   of=/tmp/myimage-backup.img \
   bs=4M \
   status=progress
```

## Restore an Image from File

```bash
# Restore from local file back to an RBD image
dd if=/tmp/myimage-backup.img \
   of=/mnt/rbd-pool/myimage-restored \
   bs=4M \
   status=progress
```

## Inspect Image Content

```bash
# Check if a specific offset contains data (detect sparse regions)
hexdump -C /mnt/rbd-pool/myimage | head -40

# Mount the image file as a loop device for filesystem access
losetup -f /mnt/rbd-pool/myimage
losetup -a  # Find the device name
mount /dev/loop0 /mnt/rbd-content

# Inspect the filesystem
ls /mnt/rbd-content
```

## Compare Two Images

```bash
# Diff two versions of an image
diff \
  <(dd if=/mnt/rbd-pool/myimage-v1 bs=4M) \
  <(dd if=/mnt/rbd-pool/myimage-v2 bs=4M) \
  > /tmp/image-diff.bin
```

## Unmount

```bash
# Unmount the FUSE filesystem
fusermount -u /mnt/rbd-pool

# Detach loop devices if used
umount /mnt/rbd-content
losetup -d /dev/loop0
```

## Summary

`rbd-fuse` exposes RBD pool images as files in a standard FUSE directory, enabling straightforward backup, restoration, and inspection of Ceph block storage volumes using familiar Unix tools. It is especially useful for ad-hoc image transfers, offline analysis, and scenarios where kernel RBD or CSI integration is unavailable.
