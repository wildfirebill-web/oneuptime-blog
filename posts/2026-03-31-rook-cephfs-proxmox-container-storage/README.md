# How to Configure CephFS for Proxmox Container Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Proxmox, CephFS, LXC, Container, Shared Storage

Description: Configure CephFS as a shared storage backend for Proxmox LXC containers, enabling container rootfs and bind mounts to be stored on a distributed filesystem.

---

Proxmox supports CephFS as a storage backend for LXC container rootfs and shared data volumes. Unlike RBD which provides block devices, CephFS provides a POSIX filesystem that can be shared across multiple Proxmox nodes simultaneously, making it ideal for shared container data.

## Use Cases for CephFS in Proxmox

- Shared configuration directories accessible by multiple containers
- Container rootfs that can be live-migrated between nodes
- Shared media storage accessible by all LXC instances
- Backup staging area with high capacity

## Step 1: Enable CephFS on Your Ceph Cluster

```bash
# Deploy the CephFS MDS
ceph fs volume create cephfs

# Verify MDS health
ceph fs status cephfs
ceph mds stat
```

## Step 2: Create Ceph User for Proxmox CephFS

```bash
ceph auth get-or-create client.proxmox-fs \
  mon 'allow r' \
  mds 'allow r, allow rw path=/' \
  osd 'allow rw tag cephfs data=cephfs, allow rw tag cephfs metadata=cephfs' \
  -o /etc/ceph/ceph.client.proxmox-fs.keyring

# Copy to all Proxmox nodes
for pve_node in pve1 pve2 pve3; do
  scp /etc/ceph/ceph.client.proxmox-fs.keyring root@${pve_node}:/etc/ceph/
done
```

## Step 3: Install CephFS Client on Proxmox Nodes

```bash
# On each Proxmox node
apt update
apt install -y ceph-common

# Verify kernel CephFS support
grep -i ceph /boot/config-$(uname -r) | grep -i "=y\|=m"
```

## Step 4: Add CephFS Storage to Proxmox

```bash
# Via CLI on a Proxmox node
pvesm add cephfs pve-cephfs \
  --monhost "mon1:6789,mon2:6789,mon3:6789" \
  --username proxmox-fs \
  --path /mnt/pve/cephfs \
  --content vztmpl,iso,backup,rootdir

# Verify storage is mounted
pvesm status | grep pve-cephfs
df -h /mnt/pve/cephfs
```

Or via the Proxmox GUI:
1. Datacenter -> Storage -> Add -> CephFS
2. Set Monitor hosts, Username, Path, and Content types

## Step 5: Create an LXC Container Using CephFS

```bash
# Create a container with rootfs on CephFS
pct create 200 pve-cephfs:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  --hostname cephfs-ct \
  --memory 512 \
  --cores 2 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage pve-cephfs \
  --rootfs pve-cephfs:8

# Start the container
pct start 200
pct status 200
```

## Step 6: Adding Shared Bind Mounts

For shared directories accessible by multiple containers:

```bash
# Create a directory in CephFS for shared data
mkdir -p /mnt/pve/cephfs/shared/app-data

# Add a bind mount to a running container
pct set 200 -mp0 /mnt/pve/cephfs/shared/app-data,mp=/shared

# Or add during creation
pct create 201 ... --mp0 /mnt/pve/cephfs/shared/app-data,mp=/shared
```

## Verifying Container Storage

```bash
# Check the container's rootfs location in CephFS
pct config 200 | grep rootfs

# Verify the actual path on the filesystem
ls /mnt/pve/cephfs/subvol-200-disk-0/
```

## Summary

CephFS in Proxmox provides a flexible shared filesystem for LXC container storage. After deploying MDS, creating a Proxmox CephFS auth user, and adding the storage via `pvesm add cephfs`, containers can have rootfs stored on CephFS and can share bind-mounted directories across nodes. Unlike RBD storage, CephFS directories are accessible from all Proxmox nodes simultaneously, making shared datasets and configuration directories straightforward to manage across a datacenter cluster.
