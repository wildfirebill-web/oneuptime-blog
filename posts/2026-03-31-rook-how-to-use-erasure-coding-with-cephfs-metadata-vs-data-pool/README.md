# How to Use Erasure Coding with CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS, Erasure Coding, Filesystem

Description: Configure CephFS with an erasure coded data pool while keeping a replicated metadata pool to balance storage efficiency with filesystem performance.

---

## Why CephFS Needs Separate Pools

CephFS uses two separate pools:
- **Metadata pool**: Stores inodes, directory entries, and file attributes
- **Data pool**: Stores actual file content

The metadata pool must use replication because it requires low-latency random access and supports RADOS object features (omap) that erasure coding does not support. The data pool can use erasure coding to save storage space.

## Architecture Overview

```text
CephFS
  |
  +-- Metadata Pool (replicated, 3x)
  |     - inodes, dentry, symlinks
  |     - small, random-access objects
  |
  +-- Data Pool (erasure coded, 4+2)
        - file data stripes
        - large, sequential-access objects
```

## Step 1 - Create the Replicated Metadata Pool

```bash
# Create metadata pool with 3x replication
ceph osd pool create cephfs-metadata 64 64 replicated
ceph osd pool application enable cephfs-metadata cephfs
```

## Step 2 - Create the EC Profile

```bash
# Create EC profile for data pool
ceph osd erasure-code-profile set cephfs-ec-profile \
  k=4 m=2 \
  plugin=jerasure \
  technique=reed_sol_van \
  stripe_unit=65536

# Verify
ceph osd erasure-code-profile get cephfs-ec-profile
```

## Step 3 - Create the EC Data Pool

```bash
# Create the data pool with the EC profile
ceph osd pool create cephfs-data 256 256 erasure cephfs-ec-profile

# Enable EC overwrites (required for CephFS data writes)
ceph osd pool set cephfs-data allow_ec_overwrites true

ceph osd pool application enable cephfs-data cephfs
```

The `allow_ec_overwrites` flag is mandatory for CephFS data pools using erasure coding. Without it, file writes will fail.

## Step 4 - Create the CephFS Filesystem

```bash
ceph fs new myfs cephfs-metadata cephfs-data
```

Verify:

```bash
ceph fs ls
ceph fs status myfs
```

## Adding Additional EC Data Pools

CephFS supports multiple data pools. Different directories can be pinned to different pools:

```bash
# Create a second data pool with different EC settings
ceph osd erasure-code-profile set cold-ec k=8 m=3 plugin=jerasure technique=reed_sol_van
ceph osd pool create cephfs-cold-data 128 128 erasure cold-ec
ceph osd pool set cephfs-cold-data allow_ec_overwrites true

# Add the pool to the filesystem
ceph fs add_data_pool myfs cephfs-cold-data

# Pin a directory to the cold pool
ceph fs set myfs file_pool myfs cephfs-cold-data
# Or use extended attributes:
setfattr -n ceph.dir.layout.pool -v cephfs-cold-data /mnt/cephfs/cold-data/
```

## Mounting the Filesystem

```bash
# Kernel client
sudo mount -t ceph mon1:6789,mon2:6789,mon3:6789:/ /mnt/cephfs \
  -o name=admin,secret=<key>

# Or using ceph-fuse
ceph-fuse -m mon1:6789 /mnt/cephfs
```

## Limitations of EC Data Pools in CephFS

Be aware of these restrictions:

```text
Limitation                | Impact
No omap support           | Client-side caching is affected
Higher write latency      | Partial file writes are slower (RMW cycles)
Stripe alignment          | Writes that cross stripe boundaries are less efficient
Recovery is slower        | EC recovery requires K chunks, unlike simple copy
```

## Performance Tuning for EC Data Pools

```bash
# Increase write buffering to reduce RMW cycles
ceph config set client fuse_big_writes true

# Increase OSD read buffer for EC recovery
ceph config set osd osd_recovery_max_single_start 8

# Monitor write performance
ceph osd pool stats cephfs-data
```

## Using Rook with EC CephFS

In Rook, configure EC pools in the CephFilesystem resource:

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
    - name: ec-data
      failureDomain: host
      erasureCoded:
        dataChunks: 4
        codingChunks: 2
  metadataServer:
    activeCount: 1
```

## Summary

CephFS with erasure coding uses a split-pool architecture: a replicated pool for metadata (where omap support and low latency are critical) and an erasure coded pool for file data (where storage efficiency matters most). The key requirement is enabling `allow_ec_overwrites` on the data pool before creating the filesystem. This setup reduces storage overhead from 3x replication to 1.5x (4+2) or less while preserving full CephFS functionality.
