# How to Configure Metadata Pool Settings for CephFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Metadata Pool, MDS, Filesystem, Kubernetes

Description: Configure the metadata pool for a Rook-Ceph CephFilesystem to control replication, failure domains, and device class placement for MDS metadata operations.

---

## Role of the Metadata Pool in CephFS

A CephFS filesystem uses two types of pools: a metadata pool and one or more data pools. The metadata pool stores filesystem directory trees, inodes, and file metadata managed by the Metadata Server (MDS). Because every file operation touches the metadata pool first, its performance characteristics have an outsized impact on filesystem latency. The metadata pool is always replicated (not erasure-coded) and is typically placed on the fastest available storage.

## Basic Metadata Pool Configuration

In Rook, the metadata pool is specified under `spec.metadataPool` in the CephFilesystem CRD:

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
      failureDomain: host
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

## Tuning the Metadata Pool for Performance

Because the metadata pool handles small random I/O (directory lookups, stat calls, and journaling), place it on SSD or NVMe OSDs using the `deviceClass` field:

```yaml
spec:
  metadataPool:
    failureDomain: host
    deviceClass: ssd
    replicated:
      size: 3
```

This generates a CRUSH rule that restricts metadata pool placement to SSD-class OSDs while data pools can use HDD OSDs, giving you low-latency metadata operations without buying all-flash storage.

## Setting Failure Domain for the Metadata Pool

The `failureDomain` on the metadata pool is independent from the data pool. Set it to match your infrastructure:

```yaml
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
```

With `size: 3` and `failureDomain: host`, the cluster tolerates losing one host without affecting filesystem availability. For zone-redundant deployments:

```yaml
spec:
  metadataPool:
    failureDomain: zone
    replicated:
      size: 3
```

## Adding Compression to the Metadata Pool

Metadata objects are small and usually compressible. Enable passive compression to reduce metadata pool footprint on flash storage:

```yaml
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
    parameters:
      compression_mode: passive
```

## Verifying Pool Settings After Deployment

After applying the CephFilesystem manifest, confirm the metadata pool is healthy:

```bash
# List CephFS pools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd lspools | grep myfs

# Check pool configuration
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get myfs-metadata all

# Verify CRUSH rule
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd crush rule dump myfs-metadata
```

The metadata pool will be named `<filesystem-name>-metadata` by convention.

## Changing Metadata Pool Settings on a Live Filesystem

You can update the metadata pool spec in the CephFilesystem CRD while the filesystem is active. Rook will reconcile the change, which may trigger a backfill if the failure domain or device class changes:

```bash
kubectl -n rook-ceph edit cephfilesystem myfs
```

Monitor the cluster health during the change:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  watch ceph status
```

## Impact of Metadata Pool Degradation

If the metadata pool loses quorum (fewer than `min_size` replicas available), the MDS daemon will evict all clients and mark the filesystem unavailable. Always ensure the metadata pool has enough redundancy and that its OSDs are on separate failure domain nodes from the data pool OSDs when possible.

## Summary

The metadata pool in a Rook CephFilesystem controls MDS performance and filesystem availability. Place it on SSD or NVMe OSDs using `deviceClass`, set an appropriate failure domain, and use at least three replicas for production workloads. Because metadata latency directly affects all filesystem operations, treating the metadata pool as a first-class performance concern pays dividends for any CephFS deployment.
