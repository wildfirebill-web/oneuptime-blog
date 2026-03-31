# How to Set Default Data Pool for CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to set and change the default data pool for a CephFS filesystem so new files are stored in the correct pool without manual per-directory configuration.

---

## What Is the Default Data Pool

When a CephFS filesystem has multiple data pools, each new file written to the filesystem goes into the default data pool unless a different pool is specified via extended attributes on the parent directory. The default data pool is set at filesystem creation time and can be changed afterward.

## Checking the Current Default Data Pool

View the current default data pool for a filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
ceph fs get myfs | grep "data_pools\|default_data_pool"
```

Or use the fs dump:

```bash
ceph fs dump --format json | jq '.filesystems[] | {name: .mdsmap.fs_name, data_pools: .mdsmap.data_pools}'
```

The first pool in the `data_pools` array is typically the default.

## Setting the Default Data Pool

Use `ceph fs set-default-data-pool` to change which pool is used by default:

```bash
ceph fs set-default-data-pool myfs myfs-ssd-data
```

Where `myfs-ssd-data` is the name of the pool you want new files to go into by default.

Verify:

```bash
ceph fs get myfs
```

The specified pool should now appear first in `data_pools`.

## Why Change the Default Data Pool

You might change the default data pool when:

- Adding an SSD-backed pool and wanting new writes to go there
- Migrating from one pool type to another (replicated to erasure-coded)
- Separating a backup or archive pool from the hot data pool

## Setting Default Pool at Filesystem Creation

When creating a filesystem with `ceph fs new`, the first data pool argument becomes the default:

```bash
# myfs-ssd-data becomes the default
ceph fs new myfs myfs-metadata myfs-ssd-data
```

Add additional pools after:

```bash
ceph fs add_data_pool myfs myfs-hdd-data
```

The default remains `myfs-ssd-data`.

## Per-Directory Pool Override

Even after setting a default pool, individual directories can override it using extended attributes. This allows coexistence of hot and cold data in the same filesystem:

```bash
# Mount the filesystem
mount -t ceph mon1:/ /mnt/myfs -o name=admin,fs=myfs

# Set the archive directory to use the HDD pool
setfattr -n ceph.dir.layout.pool -v myfs-hdd-data /mnt/myfs/archive
```

New files written to `/mnt/myfs/archive` go into `myfs-hdd-data` while files in other directories use the default pool.

## Rook CephFilesystem CRD Default Pool

In Rook's `CephFilesystem` CRD, the default data pool is the first pool listed in `dataPools`:

```yaml
spec:
  dataPools:
  - name: ssd-data      # This is the default data pool
    replicated:
      size: 3
  - name: hdd-data      # Secondary pool for cold data
    replicated:
      size: 3
      deviceClass: hdd
```

Rook creates the StorageClass with the default pool. To use the secondary pool, create a separate StorageClass:

```yaml
parameters:
  pool: myfs-hdd-data
  fsName: myfs
```

## Verifying File Pool Placement

After setting the default pool, verify that new files land in the correct pool:

```bash
# Create a test file
echo "test" > /mnt/myfs/testfile

# Check which pool the file is in
getfattr -n ceph.file.layout.pool /mnt/myfs/testfile
```

## Summary

The default CephFS data pool determines where new files are stored when no per-directory pool override is configured. Set it at creation by ordering pools in `ceph fs new`, or change it later with `ceph fs set-default-data-pool`. In Rook, the first `dataPools` entry is the default. Use per-directory `setfattr` overrides for tiered storage where different directories need different pools.
