# How to Configure Data Pool Settings for CephFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Data Pool, Filesystem, Erasure Coding, Kubernetes

Description: Configure data pool settings for Rook-Ceph CephFilesystem including replication, erasure coding, failure domains, and multiple pools for different workload types.

---

## Role of the Data Pool in CephFS

While the metadata pool stores filesystem structure and inodes, data pools store actual file content. CephFS supports multiple data pools, which lets you route different directories or files to different pools based on performance, durability, or cost requirements. Data pools can be either replicated or erasure-coded, making CephFS flexible for both latency-sensitive and capacity-optimized workloads.

## Basic Replicated Data Pool

The simplest CephFilesystem data pool configuration uses three-way replication:

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

The data pool will be named `myfs-replicated` in Ceph.

## Erasure-Coded Data Pool

For capacity efficiency on large file workloads, use an erasure-coded data pool:

```yaml
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
```

Note that the metadata pool must remain replicated. Only data pools support erasure coding in CephFS.

## Setting Device Class on the Data Pool

Route file data to HDD OSDs while keeping metadata on SSD:

```yaml
spec:
  metadataPool:
    failureDomain: host
    deviceClass: ssd
    replicated:
      size: 3
  dataPools:
    - name: hdd-data
      failureDomain: host
      deviceClass: hdd
      replicated:
        size: 3
```

## Compression on the Data Pool

Enable compression on the data pool for workloads with compressible content:

```yaml
  dataPools:
    - name: compressed-data
      failureDomain: host
      replicated:
        size: 3
      parameters:
        compression_mode: aggressive
        compression_algorithm: zstd
```

Aggressive compression attempts to compress all data. Use `passive` mode if you want compression only when the compressor believes it will succeed.

## Verifying Data Pool Configuration

After deploying the filesystem, confirm the data pool exists and is healthy:

```bash
# List pools for the filesystem
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd lspools | grep myfs

# Check pool details
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get myfs-ec-data all

# Check PG state
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg stat
```

## Checking Placement Group Autoscaling

Rook creates data pools with PG autoscaling enabled by default. Verify the autoscaler is active:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool autoscale-status
```

The `TARGET_SIZE_RATIO` and `ACTUAL_PG_COUNT` columns show whether the PG count is sized appropriately for the data stored.

## Pool-Level Quotas for Data Pools

Set a capacity limit on a data pool to prevent unbounded growth:

```yaml
  dataPools:
    - name: replicated
      failureDomain: host
      replicated:
        size: 3
      quotas:
        maxBytes: 2Ti
```

Verify quota enforcement:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get-quota myfs-replicated
```

## Summary

Data pool settings for Rook CephFilesystem control how file content is stored, protected, and compressed. Use replicated pools for low-latency random I/O, erasure-coded pools for high-capacity sequential workloads, and device class targeting to separate flash and spinning disk tiers. Multiple data pools allow a single filesystem to serve diverse workloads with appropriate storage policies per directory.
