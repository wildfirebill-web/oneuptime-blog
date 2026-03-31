# How to Load and Configure the CephFS Kernel Module

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Kernel, Module, Filesystem

Description: Load and configure the Linux CephFS kernel module (ceph.ko) for mounting distributed Ceph filesystems with kernel-native performance.

---

## The CephFS Kernel Client

Linux includes a native CephFS kernel client (`ceph.ko`) that mounts CephFS filesystems directly through the kernel VFS layer. It offers better performance and lower latency than the FUSE-based client (ceph-fuse), and supports most POSIX semantics.

## Checking Module Availability

```bash
modinfo ceph
```

Verify kernel config support:

```bash
grep CONFIG_CEPH_FS /boot/config-$(uname -r)
```

- `=y` - built in
- `=m` - loadable module

## Loading the Module

```bash
sudo modprobe ceph
```

Verify:

```bash
lsmod | grep ceph
```

CephFS also depends on the `libceph` module, which is loaded automatically.

## Persisting the Module

```bash
echo "ceph" | sudo tee /etc/modules-load.d/ceph.conf
```

## Mounting CephFS with the Kernel Client

Create the mount point and mount:

```bash
sudo mkdir -p /mnt/cephfs

# Using MON address and cephx key
sudo mount -t ceph mon1.example.com:6789,mon2.example.com:6789:/ /mnt/cephfs \
  -o name=admin,secret=AQBxxxxxx==
```

Or use a keyring file:

```bash
sudo mount -t ceph mon1.example.com:6789:/ /mnt/cephfs \
  -o name=admin,secretfile=/etc/ceph/admin.secret
```

## Persistent Mount in /etc/fstab

```fstab
mon1:6789,mon2:6789:/ /mnt/cephfs ceph name=admin,secretfile=/etc/ceph/admin.secret,_netdev,noatime 0 0
```

The `_netdev` option ensures the filesystem is mounted only after the network is up.

## Kernel Client Module Parameters

```bash
# View available parameters
ls /sys/module/ceph/parameters/

# Check current TCP nodelay setting
cat /sys/module/ceph/parameters/tcp_nodelay
```

## Useful Mount Options

| Option | Description |
|--------|-------------|
| `noatime` | Disable access time updates for performance |
| `rsize=16777216` | Read buffer size (16 MB) |
| `wsize=16777216` | Write buffer size (16 MB) |
| `caps_max=16384` | Maximum cached caps per client |
| `readdir_max_bytes=10485760` | Max bytes per readdir response |

## Summary

Load the CephFS kernel module with `modprobe ceph` and persist it in `/etc/modules-load.d/ceph.conf`. Mount CephFS using `mount -t ceph` with cephx credentials, and use `/etc/fstab` with the `_netdev` option for persistent mounts. Tune `rsize` and `wsize` mount options for workloads requiring large sequential I/O.
