# How to Mount CephFS Using FUSE (ceph-fuse)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, FUSE, Mount

Description: Learn how to mount a CephFS filesystem using ceph-fuse for userspace access with easier configuration and support on kernels without CephFS support.

---

## Overview

`ceph-fuse` is the userspace CephFS client that mounts a CephFS filesystem via the FUSE (Filesystem in Userspace) mechanism. While it has slightly higher CPU overhead than the kernel driver, `ceph-fuse` offers benefits like not requiring kernel module support, easier configuration, and access to the latest CephFS client features without waiting for kernel updates.

## When to Use ceph-fuse

Use `ceph-fuse` when:
- Your kernel does not include the CephFS module
- You need newer client features not yet in your kernel version
- Running in a container or restricted environment where kernel modules are unavailable
- Testing or development scenarios requiring easy reconfiguration

## Install ceph-fuse

```bash
# Debian/Ubuntu
apt-get install -y ceph-fuse ceph-common

# RHEL/CentOS
dnf install -y ceph-fuse ceph-common
```

## Get Cluster Configuration

Extract the cluster configuration from your Rook deployment:

```bash
# Get the Ceph configuration
kubectl -n rook-ceph get configmap rook-ceph-config -o jsonpath='{.data.ceph\.conf}'

# Get the admin keyring
kubectl -n rook-ceph get secret rook-ceph-mon -o jsonpath='{.data.ceph-secret}' | base64 -d
```

## Create Configuration Files

```bash
# Create ceph.conf
cat > /etc/ceph/ceph.conf <<EOF
[global]
fsid = <cluster-fsid>
mon_host = 192.168.1.10:6789,192.168.1.11:6789,192.168.1.12:6789
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
EOF

# Create keyring
cat > /etc/ceph/ceph.client.admin.keyring <<EOF
[client.admin]
    key = AQC...base64key...==
EOF
chmod 600 /etc/ceph/ceph.client.admin.keyring
```

## Mount with ceph-fuse

```bash
mkdir -p /mnt/cephfs

# Basic mount
ceph-fuse /mnt/cephfs

# Mount a specific filesystem (for multi-filesystem clusters)
ceph-fuse /mnt/cephfs --client_fs=cephfs

# Mount a specific directory within CephFS
ceph-fuse /mnt/cephfs --client_mountpoint=/myapp
```

## Verify the Mount

```bash
df -h /mnt/cephfs
mount | grep fuse
```

## Mount as Non-Root User

To allow non-root users to mount CephFS via FUSE:

```bash
# Add user_allow_other to /etc/fuse.conf
echo "user_allow_other" >> /etc/fuse.conf

# Mount with the allow_other option
ceph-fuse /mnt/cephfs -o allow_other
```

## Run in Foreground for Debugging

```bash
ceph-fuse /mnt/cephfs -f -d
```

The `-f` flag keeps the process in the foreground and `-d` enables debug output, useful for diagnosing connection and authentication issues.

## Unmount

```bash
fusermount -u /mnt/cephfs
# or
umount /mnt/cephfs
```

## Summary

`ceph-fuse` provides a flexible, userspace-based way to mount CephFS that works regardless of kernel version or module availability. It is especially useful in container environments, older kernels, and situations where you need the latest CephFS client features. While the kernel driver is preferred for performance-critical workloads, `ceph-fuse` is a reliable and easy-to-configure alternative for Rook-Ceph deployments.
