# How to Configure NFSv4 Pseudo-Filesystem for IPv4 Exports

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NFSv4, NFS, Pseudo-filesystem, IPv4, Exports, Linux

Description: Set up an NFSv4 pseudo-filesystem root that unifies all exports under a single namespace, configure bind mounts, and mount NFSv4 shares from IPv4 clients.

## Introduction

NFSv4 introduces a pseudo-filesystem: all exports are presented under a single root directory on the server. Clients mount paths relative to this root using a single mount point. This simplifies client configuration and enables path traversal across exports.

## Understanding NFSv4 Pseudo-Root

```
NFSv3: Each share is mounted individually
  client: mount server:/srv/data    /mnt/data
  client: mount server:/srv/backups /mnt/backups

NFSv4: One root, paths beneath it
  server: pseudo-root = /export
  server: /export/data    → bind mount → /srv/data
  server: /export/backups → bind mount → /srv/backups

  client: mount server:/ /mnt  (mounts entire pseudo-root)
  client: mount server:/data    /mnt/data    (specific path)
  client: mount server:/backups /mnt/backups
```

## Server Configuration

```bash
# Create the pseudo-root directory
sudo mkdir -p /export
sudo mkdir -p /export/data
sudo mkdir -p /export/backups
sudo mkdir -p /export/logs

# The actual data lives elsewhere — use bind mounts
sudo mkdir -p /srv/data /srv/backups /var/logs

# Bind mount actual directories into pseudo-root
sudo mount --bind /srv/data     /export/data
sudo mount --bind /srv/backups  /export/backups
sudo mount --bind /var/logs     /export/logs
```

## Making Bind Mounts Persistent

```bash
# /etc/fstab — bind mount entries
/srv/data      /export/data     none  bind  0  0
/srv/backups   /export/backups  none  bind  0  0
/var/logs      /export/logs     none  bind  0  0

# Apply fstab changes
sudo mount -a
```

## /etc/exports for NFSv4

```bash
# /etc/exports

# Export the pseudo-root (fsid=0 designates the root)
/export        10.0.0.0/24(rw,fsid=0,sync,no_subtree_check,crossmnt)

# Export individual paths under the pseudo-root
/export/data     10.0.0.0/24(rw,sync,no_subtree_check)
/export/backups  10.0.0.20(rw,sync,no_subtree_check,no_root_squash)
/export/logs     10.0.0.0/24(ro,sync,no_subtree_check)
```

```bash
# Key options for NFSv4:
# fsid=0        — designates pseudo-root (required for the root export)
# crossmnt      — allow clients to cross mount points within pseudo-root
# no_subtree_check — improves reliability

# Apply exports
sudo exportfs -ra
sudo showmount -e localhost
```

## Mounting NFSv4 from Client

```bash
# Install NFS client
sudo apt install nfs-common

# Mount specific path within pseudo-filesystem
sudo mkdir -p /mnt/data
sudo mount -t nfs4 203.0.113.10:/data /mnt/data

# Mount with explicit options
sudo mount -t nfs4 \
  -o rw,soft,_netdev,rsize=65536,wsize=65536 \
  203.0.113.10:/data /mnt/data

# Mount pseudo-root to see all exports
sudo mkdir -p /mnt/nfs
sudo mount -t nfs4 203.0.113.10:/ /mnt/nfs
ls /mnt/nfs   # Shows: data  backups  logs
```

## Persistent NFSv4 Client Mount

```bash
# /etc/fstab on client
203.0.113.10:/data     /mnt/data     nfs4  rw,soft,_netdev  0  0
203.0.113.10:/backups  /mnt/backups  nfs4  rw,soft,_netdev  0  0

sudo mount -a
```

## Verifying NFSv4

```bash
# Confirm NFSv4 is in use
sudo nfsstat -m
# Should show vers=4 for mounted filesystems

# Check NFSv4 ID mapping
sudo cat /etc/idmapd.conf
# Domain should match on both client and server

# Check NFSv4 server stats
sudo nfsstat -s

# Verify pseudo-root is exported with fsid=0
sudo exportfs -v | grep "fsid=0"
```

## Conclusion

NFSv4 pseudo-filesystems use a single export root (marked with `fsid=0`) with bind-mounted subdirectories. Set `crossmnt` on the root export to allow traversal. Clients mount paths like `server:/data` relative to the pseudo-root. This simplifies NFSv4 client configuration compared to NFSv3's per-export mounts.
