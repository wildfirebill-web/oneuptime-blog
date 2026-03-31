# How to Manage Data Pools for CephFS (Add and Remove)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, CephFS

Description: Learn how to add and remove data pools from a CephFS filesystem to support tiered storage, erasure coding, or capacity expansion in Rook clusters.

---

## CephFS Data Pool Overview

CephFS supports multiple data pools per filesystem. This allows you to implement storage tiering - for example, storing hot data in an SSD-backed replicated pool and cold data in an HDD-backed erasure-coded pool. Clients can select which data pool to use on a per-directory basis.

## Adding a Data Pool

Create a new pool and add it to an existing CephFS filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

```bash
# Create a new pool (e.g., an HDD-backed erasure-coded pool for cold data)
ceph osd pool create myfs-cold-data 64 erasure
ceph osd pool set myfs-cold-data allow_ec_overwrites true

# Add the pool to the filesystem
ceph fs add_data_pool myfs myfs-cold-data
```

Verify the pool was added:

```bash
ceph fs get myfs | grep data_pools
```

Expected output:

```text
data_pools    [2 3 ]
```

Both pool IDs are now listed.

## Listing Data Pools

View all data pools for a filesystem:

```bash
ceph fs ls
```

Sample output:

```text
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-data myfs-cold-data ]
```

## Setting a Directory's Data Pool

Once a pool is added, configure specific directories to use it via extended attributes. Mount the filesystem and use `setfattr`:

```bash
# Set a directory to use the cold data pool
setfattr -n ceph.dir.layout.pool -v myfs-cold-data /mnt/myfs/archives
```

Or use the Ceph admin tool:

```bash
ceph fs set-default-data-pool myfs myfs-data
```

## Adding a Data Pool via Rook CephFilesystem CRD

In Rook, update the `CephFilesystem` spec to add a new data pool:

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
  - name: replicated
    replicated:
      size: 3
  - name: cold
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  metadataServer:
    activeCount: 1
```

Apply the updated spec:

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook creates the new pool and runs `ceph fs add_data_pool` automatically.

## Removing a Data Pool

Before removing a data pool, ensure no files are stored in it. Use the default pool or migrate data first.

Remove the pool from the filesystem:

```bash
ceph fs rm_data_pool myfs myfs-cold-data
```

Then optionally delete the underlying pool:

```bash
ceph osd pool delete myfs-cold-data myfs-cold-data --yes-i-really-really-mean-it
```

Note: You cannot remove the default data pool. Set a different pool as default first if needed.

## Verifying Pool Removal

Confirm the pool was removed from the filesystem:

```bash
ceph fs get myfs | grep data_pools
```

Only the remaining pools should be listed.

## Summary

CephFS supports multiple data pools for storage tiering. Add pools with `ceph fs add_data_pool` and remove them with `ceph fs rm_data_pool`. In Rook, manage data pools declaratively via the `dataPools` array in the `CephFilesystem` CRD. Set per-directory pool layouts using extended attributes to route specific workloads (hot/cold data, compliance archives) to appropriate storage tiers.
