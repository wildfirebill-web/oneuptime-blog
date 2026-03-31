# How to Enable fscrypt for CephFS Encryption

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, fscrypt, Encryption, Security

Description: Learn how to enable fscrypt for CephFS directory-level encryption in Ceph, configure encryption policies on directories, and manage encryption keys for filesystem data.

---

## What is fscrypt for CephFS?

fscrypt is a Linux kernel feature that provides per-directory encryption for filesystems. CephFS supports fscrypt natively, allowing you to encrypt specific directories within a CephFS mount without encrypting the entire OSD. This is useful for multi-tenant environments where different tenants need isolated encryption keys.

## Prerequisites

- Linux kernel 5.15+ (for stable CephFS fscrypt support)
- CephFS with `mds_max_mdsmap_epochs` configured
- `fscrypt` userspace tool installed
- CephFS formatted with encryption support

Install the fscrypt tool:

```bash
apt-get install fscrypt
# or
yum install fscrypt
```

## Enabling fscrypt on a CephFS Mount

### Step 1: Mount CephFS

```bash
mount -t ceph mon1:6789,mon2:6789,mon3:6789:/ /mnt/cephfs \
  -o name=admin,secret=<admin-keyring>
```

### Step 2: Format the Filesystem for fscrypt

```bash
fscrypt setup /mnt/cephfs
```

This creates a `.fscrypt` directory at the root of the mount point.

### Step 3: Create an Encryption Policy

Create a directory and apply an encryption policy:

```bash
mkdir /mnt/cephfs/tenant-a
fscrypt encrypt /mnt/cephfs/tenant-a
```

You will be prompted to create or select a protector (key source). Options include:
- `pam_passphrase` - user login passphrase
- `custom_passphrase` - manually entered passphrase
- `raw_key` - raw key file

### Step 4: Using a Raw Key File

For automation, use a raw key:

```bash
# Generate a 32-byte key
dd if=/dev/urandom of=/etc/ceph/tenant-a.key bs=32 count=1

fscrypt encrypt /mnt/cephfs/tenant-a \
  --source=raw_key \
  --key=/etc/ceph/tenant-a.key \
  --name="tenant-a-key"
```

## Verifying Encryption Status

```bash
fscrypt status /mnt/cephfs/tenant-a
```

Expected output:
```json
"/mnt/cephfs/tenant-a": encrypted with fscrypt
Policy:   abc123...
Unlocked: Yes
```

## Locking and Unlocking Directories

Lock a directory (removes key from kernel keyring):

```bash
fscrypt lock /mnt/cephfs/tenant-a
```

Unlock with the key:

```bash
fscrypt unlock /mnt/cephfs/tenant-a \
  --key=/etc/ceph/tenant-a.key
```

## CephFS Encryption Policy via Client Config

For Rook-managed CephFS, mount options can be configured in the CSI driver config to apply fscrypt automatically to PVC-backed directories.

## Summary

fscrypt for CephFS provides directory-level encryption that supports per-tenant key isolation without encrypting the underlying OSD devices. Enable it by running `fscrypt setup` on a mounted CephFS volume, then apply encryption policies to specific directories using `fscrypt encrypt`. Use raw key files for automation and integrate key storage with a secrets manager for production deployments.
