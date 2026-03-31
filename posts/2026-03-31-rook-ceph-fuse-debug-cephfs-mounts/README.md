# How to Use ceph-fuse for Debugging CephFS Mounts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, ceph-fuse, Debugging, Filesystem

Description: Use ceph-fuse to mount and debug CephFS filesystems with detailed logging, useful for diagnosing mount failures, permission issues, and metadata problems.

---

## Why ceph-fuse for Debugging

The kernel CephFS client (`mount -t ceph`) has limited logging capabilities. `ceph-fuse` runs in userspace and provides verbose debug output, making it superior for diagnosing mount failures, MDS issues, and client permission problems.

## Basic ceph-fuse Mount

```bash
# Install ceph-fuse
apt-get install -y ceph-fuse   # Debian/Ubuntu
yum install -y ceph-fuse       # RHEL/CentOS

# Create mount point
mkdir -p /mnt/cephfs-debug

# Mount with debug logging
ceph-fuse \
  --conf /etc/ceph/ceph.conf \
  --keyring /etc/ceph/ceph.client.admin.keyring \
  --debug-fuse 10 \
  --debug-ms 1 \
  /mnt/cephfs-debug
```

## Get Config from Rook

Extract the keyring and config from Rook secrets:

```bash
# Get the admin keyring
kubectl -n rook-ceph get secret rook-ceph-admin-keyring \
  -o jsonpath='{.data.keyring}' | base64 -d > /tmp/keyring

# Get the Ceph config
kubectl -n rook-ceph get cm rook-ceph-config \
  -o jsonpath='{.data.ceph\.conf}' > /tmp/ceph.conf
```

## Mount a Specific CephFS Path

```bash
# Mount a specific subdirectory (useful for tenant isolation debugging)
ceph-fuse \
  --conf /tmp/ceph.conf \
  --keyring /tmp/keyring \
  --client-mountpoint /myapp-data \
  /mnt/cephfs-debug

# Verify the mount
ls -la /mnt/cephfs-debug
df -h /mnt/cephfs-debug
```

## Enable Maximum Debug Verbosity

```bash
# Debug all CephFS client subsystems
ceph-fuse \
  --conf /tmp/ceph.conf \
  --keyring /tmp/keyring \
  --debug-fuse 20 \
  --debug-mds-client 20 \
  --debug-objecter 5 \
  --debug-ms 1 \
  /mnt/cephfs-debug 2>&1 | tee /tmp/ceph-fuse-debug.log
```

## Debug MDS Cap Issues

```bash
# Mount and check cap stats
ceph-fuse --conf /tmp/ceph.conf /mnt/cephfs-debug &

# Query client session from MDS
ceph tell mds.* client ls

# Check cap count for the fuse client
ceph tell mds.0 dump_cache | grep caps
```

## Debug Permission Denied Errors

```bash
# Mount with specific user credentials
ceph-fuse \
  --conf /tmp/ceph.conf \
  --name client.myapp \
  --keyring /tmp/myapp.keyring \
  /mnt/cephfs-debug

# Check what paths the user has access to
ceph auth get client.myapp
```

## Unmount and Cleanup

```bash
# Unmount gracefully
fusermount -u /mnt/cephfs-debug

# Force unmount if stuck
fusermount -u -z /mnt/cephfs-debug
```

## Summary

`ceph-fuse` provides userspace CephFS mounting with rich debug logging, making it the preferred tool for diagnosing CephFS issues like mount failures, MDS connection problems, and capability exhaustion. The verbose output from `--debug-fuse` and `--debug-mds-client` flags exposes client-MDS interactions that are invisible in kernel mount logs.
