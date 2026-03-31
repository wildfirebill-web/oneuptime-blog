# How to Configure Data Deduplication in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Deduplication, Dedup, Storage Efficiency, BlueStore, Object Storage

Description: Configure data deduplication in Ceph to reduce storage consumption for workloads with redundant data such as VM images, backups, and object storage archives.

---

## Ceph Deduplication Overview

Ceph supports deduplication at two levels:
1. **RGW object-level dedup**: The `rgw_dedup` module deduplicates objects stored in the RADOS Gateway by content fingerprinting
2. **Pool-level dedup (experimental)**: Inline deduplication for RADOS pools (still maturing in Ceph Reef/Squid)

RGW-level deduplication is the most stable and widely used approach for object storage workloads.

## Enabling RGW Object Deduplication

The RGW dedup process runs as a background job. Enable it on the zone:

```bash
# Enable dedup configuration in the zone
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin zone modify \
  --rgw-zone default \
  --dedup-chunk-algo fixed \
  --dedup-chunk-size 65536  # 64KB chunks

# Start the dedup process
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin dedup start
```

## Configuring the Dedup Pool

Dedup requires a separate chunk pool where unique data chunks are stored:

```yaml
# Create a dedicated chunk pool for dedup
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: dedup-chunk-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    pg_autoscale_mode: "on"
    compression_mode: aggressive  # Compress deduplicated chunks further
```

```bash
# Configure RGW to use the chunk pool
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin zone modify \
  --rgw-zone default \
  --dedup-pool rook-ceph.dedup-chunk-pool.data
```

## Checking Deduplication Savings

```bash
# View dedup statistics
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin dedup stats

# Example output shows:
# Processed: 1.2 TiB
# Deduplicated: 340 GiB (28% savings)
# Unique chunks: 890,234
```

## Pool-Level Dedup Configuration (Experimental)

For RADOS pools (not RGW), experimental inline dedup can be configured:

```bash
# Enable dedup on a pool (Ceph 17+ - use with caution)
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set mypool dedup_tier mypool-dedup-chunk-pool

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set mypool dedup_chunk_algorithm fastcdc

kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set mypool dedup_chunk_size 131072  # 128KB
```

## Workloads That Benefit from Dedup

| Workload | Expected Savings |
|----------|-----------------|
| VM images (same OS base) | 40-70% |
| Backup repositories | 50-80% |
| Container image layers | 30-60% |
| Document archives | 20-40% |
| Video/media files | Near 0% |
| Encrypted data | Near 0% |

## Enabling Compression Alongside Dedup

Combine dedup with compression for maximum savings:

```bash
# Enable compression on the main data pool
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set my-object-store-data-pool \
  compression_mode aggressive

# Verify compression ratio
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph df detail | grep -A3 "my-object-store"
```

## Monitoring Dedup Efficiency

```promql
# Pool compression ratio (proxy for dedup+compression effectiveness)
ceph_pool_compress_under_bytes{name="my-pool"} /
ceph_pool_compress_bytes_used{name="my-pool"}
```

## Summary

Ceph data deduplication reduces storage consumption for workloads with redundant content. RGW-level dedup via the `rgw_dedup` module is the most production-ready approach, using content fingerprinting to identify and deduplicate identical chunks across objects. Combined with pool-level compression, organizations storing VM images, backups, or document archives can see 50-70% effective storage savings. Encrypted and media workloads see negligible benefit and should not be configured for dedup.
