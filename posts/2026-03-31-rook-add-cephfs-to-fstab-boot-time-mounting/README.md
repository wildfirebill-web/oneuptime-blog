# How to Add CephFS to fstab for Boot-Time Mounting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Mount, Linux

Description: Learn how to configure persistent CephFS mounts using fstab or systemd mount units for automatic boot-time mounting from Rook-Ceph clusters.

---

## Overview

Configuring CephFS to mount automatically at boot time is essential for servers that need persistent access to shared storage. On Linux, this is done by adding the appropriate entry to `/etc/fstab` or creating a systemd mount unit. Both approaches have trade-offs in terms of flexibility and boot dependency management.

## Prerequisites

Before adding CephFS to fstab, complete the prerequisites:
- `ceph-common` installed
- `/etc/ceph/ceph.conf` configured with monitor addresses
- `/etc/ceph/ceph.client.admin.keyring` in place with correct permissions
- Mount point directory created

```bash
mkdir -p /mnt/cephfs
```

## fstab Entry for Kernel CephFS Driver

Add the following entry to `/etc/fstab`:

```text
192.168.1.10:6789,192.168.1.11:6789,192.168.1.12:6789:/  /mnt/cephfs  ceph  name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring,noatime,_netdev  0  0
```

Key options explained:

```text
name=admin          - CephX client name (without the "client." prefix)
secretfile=...      - Path to the keyring file
noatime             - Skip access time updates for performance
_netdev             - Tells systemd this mount requires the network
0 0                 - Do not dump or fsck at boot
```

## fstab Entry Using Inline Secret

Alternatively, embed the key directly (less secure):

```text
192.168.1.10:6789:/  /mnt/cephfs  ceph  name=admin,secret=AQC...key...==,noatime,_netdev  0  0
```

## Test the fstab Entry

Before rebooting, test the mount with:

```bash
mount -a -T /etc/fstab
```

Or test specifically:

```bash
mount /mnt/cephfs
df -h /mnt/cephfs
```

## Using a Systemd Mount Unit

For more control over mount ordering and dependencies, use a systemd mount unit. The unit file name must match the mount point path:

Create `/etc/systemd/system/mnt-cephfs.mount`:

```text
[Unit]
Description=CephFS Mount
After=network-online.target
Wants=network-online.target

[Mount]
What=192.168.1.10:6789,192.168.1.11:6789:/
Where=/mnt/cephfs
Type=ceph
Options=name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring,noatime

[Install]
WantedBy=multi-user.target
```

Enable and start the unit:

```bash
systemctl daemon-reload
systemctl enable mnt-cephfs.mount
systemctl start mnt-cephfs.mount
systemctl status mnt-cephfs.mount
```

## Handle Mount Failures Gracefully

Add `nofail` to fstab entries to prevent boot failure if CephFS is unavailable:

```text
192.168.1.10:6789:/  /mnt/cephfs  ceph  name=admin,secretfile=/etc/ceph/ceph.client.admin.keyring,noatime,_netdev,nofail  0  0
```

## Summary

Adding CephFS to `/etc/fstab` or a systemd mount unit enables automatic boot-time mounting of Rook-Ceph shared storage. The `_netdev` option ensures the mount waits for network availability, `nofail` prevents boot failures if the cluster is temporarily unavailable, and `noatime` reduces unnecessary write traffic. Systemd mount units offer finer control over dependencies and are recommended for production servers where boot order reliability matters.
