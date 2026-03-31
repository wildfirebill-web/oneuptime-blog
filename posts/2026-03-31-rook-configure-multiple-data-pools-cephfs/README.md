# How to Configure Multiple Data Pools for CephFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Data Pool, Filesystem, Kubernetes, Storage Tier

Description: Configure multiple data pools in a Rook CephFilesystem to support tiered storage, allowing different directories to use different performance or durability pools.

---

## Why Use Multiple Data Pools

CephFS supports associating multiple data pools with a single filesystem. Each pool can have different replication settings, erasure-coding profiles, device classes, or compression settings. By mapping directories to specific pools using CephFS layout policies, you can implement storage tiers within a single filesystem namespace: fast NVMe pools for hot data, replicated SSD pools for warm data, and erasure-coded HDD pools for cold archives.

## Defining Multiple Data Pools in Rook

Specify multiple entries in the `dataPools` list of the CephFilesystem CRD. The first pool in the list becomes the default for new files:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    deviceClass: ssd
    replicated:
      size: 3
  dataPools:
    - name: nvme-hot
      failureDomain: host
      deviceClass: nvme
      replicated:
        size: 3
    - name: ssd-warm
      failureDomain: host
      deviceClass: ssd
      replicated:
        size: 3
    - name: hdd-cold
      failureDomain: host
      deviceClass: hdd
      erasureCoded:
        dataChunks: 4
        codingChunks: 2
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Rook creates three pools: `myfs-nvme-hot`, `myfs-ssd-warm`, and `myfs-hdd-cold`.

## Verifying Pool Creation

Confirm all three pools were created successfully:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd lspools | grep myfs

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs ls
```

The `ceph fs ls` output shows the filesystem with all data pools listed.

## Setting the Default Data Pool

The first pool in the `dataPools` list is automatically set as the default. To verify or change the default pool:

```bash
# Check current default data pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs get myfs | grep data_pools

# Change the default pool
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs set myfs default_data_pool myfs-ssd-warm
```

## Routing Directories to Specific Pools

Use CephFS layout attributes to assign a directory to a specific pool. This must be done from inside a pod that has the filesystem mounted:

```bash
# Exec into a pod with the filesystem mounted
kubectl exec -it mypod -- bash

# Set the layout on a directory to use the cold pool
setfattr -n ceph.dir.layout.pool -v myfs-hdd-cold /mnt/cephfs/archive

# Verify the layout
getfattr -n ceph.dir.layout /mnt/cephfs/archive
```

New files created inside `/mnt/cephfs/archive` will be written to the `myfs-hdd-cold` pool. Files created before the layout change are not retroactively moved.

## Adding a New Data Pool to an Existing Filesystem

To add a pool to a running filesystem without disruption, add a new entry to the `dataPools` list and apply the updated CephFilesystem manifest:

```yaml
  dataPools:
    - name: nvme-hot
      ...
    - name: ssd-warm
      ...
    - name: hdd-cold
      ...
    - name: compressed-archive
      failureDomain: host
      deviceClass: hdd
      replicated:
        size: 3
      parameters:
        compression_mode: aggressive
```

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook reconciles the change and adds the new pool to the filesystem.

## Removing a Data Pool

To remove a pool, delete it from the `dataPools` list and apply the manifest. Rook will attempt to remove the pool from Ceph. Ensure no files are using the pool before removal, as removing a pool with data causes data loss:

```bash
# Check if the pool has data before removing
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rados -p myfs-hdd-cold ls | head -10
```

If the pool is empty, proceed with the removal.

## Summary

Multiple data pools in a Rook CephFilesystem enable tiered storage within a single shared namespace. Define each pool with its own device class, replication strategy, and compression settings in the `dataPools` list, then use CephFS layout attributes to route directory trees to specific pools. This pattern is powerful for mixed workloads where hot and cold data coexist but need different storage policies.
