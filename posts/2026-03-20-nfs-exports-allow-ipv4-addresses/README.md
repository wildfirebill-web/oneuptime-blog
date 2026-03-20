# How to Configure NFS Exports to Allow Specific IPv4 Addresses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: NFS, IPv4, Exports, Access Control, Linux, File Sharing

Description: Configure /etc/exports to allow NFS access from specific IPv4 addresses and subnets, set mount options, and reload the NFS server to apply changes.

## Introduction

NFS access control is primarily managed through `/etc/exports`. Each exported directory specifies which clients (by IP address, hostname, or subnet) can mount it, and with what options. Proper exports configuration prevents unauthorized access to shared filesystems.

## Basic /etc/exports Syntax

```bash
# /etc/exports format:
# /path/to/share  client(options) [client2(options2)]

# Allow a single IPv4 address
/srv/data  203.0.113.20(rw,sync,no_subtree_check)

# Allow multiple individual IPs
/srv/data  10.0.0.5(rw,sync) 10.0.0.6(rw,sync) 10.0.0.7(ro,sync)

# Read-only for one IP, read-write for another
/srv/backups  203.0.113.20(ro,sync) 10.0.0.5(rw,sync,no_root_squash)
```

## Common Export Options

| Option | Meaning |
|---|---|
| `rw` | Read-write access |
| `ro` | Read-only access |
| `sync` | Write to disk before acknowledging |
| `async` | Faster but less safe writes |
| `no_subtree_check` | Disable subtree checking (recommended) |
| `root_squash` | Map root to anonymous user (default) |
| `no_root_squash` | Keep root as root (trusted clients only) |
| `all_squash` | Map all users to anonymous |

## Configuring Exports for Specific IPs

```bash
# /etc/exports

# Web servers: read-only content
/var/www/html  10.0.0.10(ro,sync,no_subtree_check) \
               10.0.0.11(ro,sync,no_subtree_check)

# Database backup server: read-write
/srv/db-backups  10.0.0.20(rw,sync,no_subtree_check,no_root_squash)

# Monitoring server: read-only logs
/var/log  10.0.0.30(ro,sync,no_subtree_check)
```

## Applying Changes

```bash
# Install NFS server
sudo apt install nfs-kernel-server

# Create shared directory
sudo mkdir -p /srv/data
sudo chown nobody:nogroup /srv/data
sudo chmod 755 /srv/data

# Edit exports
sudo nano /etc/exports

# Apply changes without restarting
sudo exportfs -ra

# View active exports
sudo exportfs -v

# Restart NFS server
sudo systemctl restart nfs-kernel-server
```

## Verifying Exports

```bash
# On the NFS server: list all active exports
sudo showmount -e localhost
# or
sudo exportfs -v

# From an NFS client: see what's available
showmount -e 203.0.113.10

# Check NFS logs
sudo journalctl -u nfs-kernel-server -f

# Verify an IP can see the exports
sudo showmount -e 203.0.113.10
# /srv/data  10.0.0.5,10.0.0.6
```

## Security Best Practices

```bash
# /etc/exports — security-focused configuration

# Use sync to avoid data loss on server crash
/srv/data  10.0.0.0/24(rw,sync,no_subtree_check,root_squash)

# Never export / (root filesystem)
# Never export home directories without careful thought

# Restrict to loopback for local-only access
/srv/local-only  127.0.0.1(rw,sync,no_subtree_check)

# Verify no world-accessible exports exist:
sudo exportfs -v | grep -v "10\.\|192\.168\.\|127\."
```

## Conclusion

NFS access control via `/etc/exports` specifies which IPv4 addresses and subnets can mount each share. Use `rw`/`ro` for permissions, `sync` for data integrity, `no_subtree_check` for reliability, and `root_squash` for security. Apply changes with `exportfs -ra` and verify with `showmount -e localhost`.
