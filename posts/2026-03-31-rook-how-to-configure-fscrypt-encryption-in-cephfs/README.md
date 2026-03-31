# How to Configure fscrypt Encryption in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Encryption, fscrypt, Security

Description: Configure fscrypt encryption in CephFS to encrypt file data and metadata at the filesystem level, providing per-directory encryption with kernel-integrated key management.

---

## What Is fscrypt in CephFS

fscrypt is a Linux kernel framework for filesystem-level encryption. CephFS supports fscrypt starting with Ceph Quincy (17.x), allowing directories to be encrypted using keys managed by the kernel's keyring subsystem.

Unlike Ceph-native encryption (which encrypts at the OSD layer), fscrypt encrypts data at the client side before it reaches the network, providing true end-to-end encryption.

## Prerequisites

- Ceph Quincy (17.2) or later
- Linux kernel 5.4 or later (for fscrypt v2 API)
- `fscryptctl` or `fscrypt` user-space tool installed on clients
- CephFS kernel client (not ceph-fuse for initial setup)

```bash
# Install fscryptctl on client
apt-get install fscryptctl

# Or build from source
git clone https://github.com/google/fscryptctl
cd fscryptctl && make
```

## Step 1 - Enable fscrypt on the CephFS Filesystem

The cluster must have fscrypt enabled:

```bash
# Enable fscrypt feature
ceph fs set myfs fscrypt true

# Verify
ceph fs get myfs | grep fscrypt
```

## Step 2 - Mount CephFS with fscrypt Support

Mount the filesystem with the kernel client:

```bash
# Mount with kernel client
sudo mount -t ceph mon1:6789,mon2:6789:/ /mnt/cephfs \
  -o name=admin,secretfile=/etc/ceph/admin.keyring

# Verify mount
mount | grep ceph
```

## Step 3 - Initialize fscrypt on the Mount Point

```bash
# Initialize fscrypt on the mounted filesystem
fscryptctl setup /mnt/cephfs

# This creates /.fscrypt directory at the mount root
ls /mnt/cephfs/.fscrypt/
```

## Step 4 - Create an Encryption Policy

Generate an encryption key and create a policy:

```bash
# Generate a random 512-bit key (for AES-256-XTS)
dd if=/dev/urandom bs=64 count=1 2>/dev/null | base64 > /secure/location/mykey.b64

# Add the key to the kernel keyring
key_descriptor=$(fscryptctl add_key /mnt/cephfs < /secure/location/mykey.b64)
echo "Key descriptor: $key_descriptor"
```

## Step 5 - Apply Encryption to a Directory

```bash
# Create the directory to encrypt
mkdir /mnt/cephfs/encrypted-dir

# Apply the encryption policy
fscryptctl set_policy $key_descriptor /mnt/cephfs/encrypted-dir

# Verify the policy is set
fscryptctl get_policy /mnt/cephfs/encrypted-dir
```

## Step 6 - Verify Encryption

Files written to the encrypted directory are encrypted on disk:

```bash
# Write a test file
echo "Secret data" > /mnt/cephfs/encrypted-dir/test.txt

# Verify encryption is active
cat /mnt/cephfs/encrypted-dir/test.txt
# Output: Secret data (decrypted because key is in keyring)

# Remove key and verify data is inaccessible
fscryptctl remove_key /mnt/cephfs < /secure/location/mykey.b64
ls /mnt/cephfs/encrypted-dir/
# Shows encrypted filenames (garbled names)
cat /mnt/cephfs/encrypted-dir/<garbled-name>
# Error: key unavailable
```

## Encryption Algorithms

fscrypt supports:

```text
Contents Encryption:  AES-256-XTS (recommended)
Filenames Encryption: AES-256-CTS-CBC or AES-256-HCTR2
```

```bash
# Set a policy with explicit cipher
fscryptctl set_policy --contents=AES-256-XTS --filenames=AES-256-CTS-CBC \
  $key_descriptor /mnt/cephfs/encrypted-dir
```

## Key Protectors and Key Derivation

For production use, integrate with PAM or a key management system:

```bash
# Use the high-level fscrypt tool (wraps fscryptctl)
fscrypt setup /mnt/cephfs
fscrypt encrypt /mnt/cephfs/encrypted-dir

# This prompts for a passphrase and handles key derivation
```

## Rook and fscrypt

In Rook, enable fscrypt on the CephFilesystem resource:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
```

After Rook creates the filesystem, enable fscrypt via the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set myfs fscrypt true
```

## Limitations

```text
- Kernel client only (no fscrypt support in ceph-fuse)
- Per-directory granularity (not per-file)
- Key must be present in kernel keyring to access encrypted files
- Snapshots of encrypted directories store encrypted data
- NFS re-export of encrypted CephFS is not supported
```

## Summary

fscrypt in CephFS provides client-side per-directory encryption using the Linux kernel's encryption framework. Enabling it requires setting `fscrypt true` on the filesystem, mounting with the kernel client, and using `fscryptctl` to assign encryption policies to directories. Data and filenames are encrypted before leaving the client, providing strong security even if OSD storage is compromised. Key management via the kernel keyring or `fscrypt` tool enables per-user or per-session access control to encrypted directories.
