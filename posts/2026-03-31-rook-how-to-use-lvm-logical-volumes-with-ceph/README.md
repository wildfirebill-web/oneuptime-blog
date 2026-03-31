# How to Use LVM Logical Volumes with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, LVM, OSD, Storage

Description: Configure Ceph OSDs to use LVM logical volumes for flexible disk partitioning, thin provisioning, and co-locating BlueStore WAL and DB on faster devices.

---

## Why Use LVM with Ceph

Ceph OSDs can be deployed on raw block devices or on LVM logical volumes (LVs). Using LVM provides:

- Flexible partition sizing across physical disks
- Ability to place BlueStore WAL and DB on separate (faster) devices
- Thin provisioning for testing environments
- Logical grouping of multiple physical disks into a single OSD

## Understanding BlueStore Components on LVM

BlueStore (the default OSD backend) has three storage components:
- **Data** - stores the actual object data (typically HDD)
- **WAL** (Write-Ahead Log) - tiny, sequential writes (benefits from NVMe)
- **DB** - RocksDB metadata (benefits from SSD/NVMe)

## Creating LVM Volumes for Ceph OSDs

### Single Disk LVM OSD

Create a volume group and logical volume on a single disk:

```bash
# Create volume group
sudo vgcreate ceph-vg-data /dev/sdb

# Create logical volume for OSD data
sudo lvcreate -l 100%FREE -n ceph-lv-data ceph-vg-data
```

### Separate WAL/DB on Faster Storage

```bash
# HDD for data
sudo vgcreate ceph-data-vg /dev/sdb
sudo lvcreate -l 100%FREE -n ceph-data ceph-data-vg

# NVMe for WAL and DB
sudo vgcreate ceph-fast-vg /dev/nvme0n1
sudo lvcreate -L 10G -n ceph-wal ceph-fast-vg
sudo lvcreate -L 50G -n ceph-db ceph-fast-vg
```

## Configuring Rook to Use LVM Volumes

In Rook, specify LVM volumes in the `CephCluster` storage spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  storage:
    useAllNodes: false
    nodes:
      - name: "worker-1"
        devices:
          - name: "/dev/ceph-data-vg/ceph-data"
            config:
              osdsPerDevice: "1"
              metadataDevice: "/dev/ceph-fast-vg/ceph-db"
```

Apply:

```bash
kubectl apply -f cephcluster.yaml
```

## Preparing LVM Volumes Before Rook Deployment

Ensure LVM metadata is clean before Rook discovers devices:

```bash
# Wipe any existing LVM metadata
sudo wipefs -a /dev/sdb
sudo dd if=/dev/zero of=/dev/sdb bs=1M count=100

# Create fresh LVM structure
sudo pvcreate /dev/sdb
sudo vgcreate ceph-data-vg /dev/sdb
sudo lvcreate -l 100%FREE -n ceph-data ceph-data-vg
```

## Verifying LVM OSD Deployment

After Rook creates OSDs on LVM volumes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
```

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-osd
```

Check which device an OSD is using:

```bash
kubectl -n rook-ceph exec -it <osd-pod> -- ceph-volume lvm list
```

Output:

```text
====== osd.0 =======
  [block]       /dev/ceph-data-vg/ceph-data
  type          block
  osd id        0
  cluster fsid  abc123...
  [db]          /dev/ceph-fast-vg/ceph-db
  [wal]         /dev/ceph-fast-vg/ceph-wal
```

## Thin Provisioning for Testing

For lab environments, create thin-provisioned LVs to simulate more OSDs than physical space allows:

```bash
# Create a thin pool
sudo vgcreate ceph-thin-vg /dev/sdb
sudo lvcreate -L 100G --thinpool ceph-thin-pool ceph-thin-vg

# Create thin LVs (each appears as 20GB but shares underlying space)
sudo lvcreate -V 20G --thin -n ceph-osd-0 ceph-thin-vg/ceph-thin-pool
sudo lvcreate -V 20G --thin -n ceph-osd-1 ceph-thin-vg/ceph-thin-pool
sudo lvcreate -V 20G --thin -n ceph-osd-2 ceph-thin-vg/ceph-thin-pool
```

**Note:** Thin provisioning is only suitable for testing. Do not use it in production.

## Removing LVM OSDs

When decommissioning an LVM-based OSD:

```bash
# Remove from Ceph cluster first
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd out osd.0
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd purge osd.0 --yes-i-really-mean-it

# Then clean LVM metadata
sudo lvremove /dev/ceph-data-vg/ceph-data
sudo vgremove ceph-data-vg
sudo pvremove /dev/sdb
```

## Summary

Using LVM logical volumes with Ceph provides flexibility to separate BlueStore WAL and DB metadata onto faster NVMe or SSD devices while keeping bulk data on cheaper HDDs. Configure Rook's `CephCluster` storage spec to point at LVM device paths, ensuring volumes are clean and properly tagged. In production environments, separate WAL and DB placement on fast storage can significantly improve OSD write performance.
