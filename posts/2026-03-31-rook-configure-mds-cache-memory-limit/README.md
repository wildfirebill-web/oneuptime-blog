# How to Configure MDS Cache Memory Limit in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, MDS, CephFS, Memory, Performance, Kubernetes

Description: Learn how to configure the CephFS MDS cache memory limit to improve metadata performance while preventing OOM kills in Kubernetes environments.

---

## Understanding MDS Cache

The Ceph Metadata Server (MDS) maintains an in-memory cache of filesystem metadata including directory entries, inode information, and file attributes. A larger cache means fewer reads from the metadata pool, resulting in lower latency for metadata-intensive workloads like directory listings and file lookups.

By default, the MDS cache limit is 4GB. In memory-constrained environments this can cause OOM kills, while on memory-rich nodes you may want to increase it for better performance.

## Setting the MDS Cache Memory Limit

Configure the cache limit in the CephFilesystem spec:

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
  - replicated:
      size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        memory: "8Gi"
        cpu: "2"
      limits:
        memory: "12Gi"
        cpu: "4"
    config:
      MDS_CACHE_MEMORY_LIMIT: "8589934592"
```

The `MDS_CACHE_MEMORY_LIMIT` value is in bytes (8589934592 = 8GB).

## Setting Cache Limit Via Ceph Config

You can also set this at the Ceph configuration level:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 8589934592

# Verify the setting
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config get mds mds_cache_memory_limit
```

## Checking Current Cache Usage

Monitor how much cache the MDS is actually using:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mds stat

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs status myfs
```

For detailed cache statistics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph daemon mds.$(ceph mds stat | awk '/active/ {print $1}') cache status
```

## Tuning Cache Pressure Settings

Adjust when the MDS starts aggressively trimming the cache:

```bash
# Start trimming when cache is 90% full (default 0.9)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_trim_threshold 0.9

# Trim interval in seconds (default 5)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_trim_decay_rate 1.0
```

## Sizing Recommendations

Guidelines for MDS cache sizing:

```bash
# For small filesystems (< 10M files): 4GB cache
ceph config set mds mds_cache_memory_limit 4294967296

# For medium filesystems (10M - 100M files): 8-16GB
ceph config set mds mds_cache_memory_limit 8589934592

# For large filesystems (> 100M files): 16-32GB
ceph config set mds mds_cache_memory_limit 17179869184
```

Always set the Kubernetes memory limit to at least 20% above the MDS cache limit to account for other MDS memory usage.

## Summary

The MDS cache memory limit is a critical tuning parameter that directly affects CephFS metadata performance. Set it in the CephFilesystem CRD via the `MDS_CACHE_MEMORY_LIMIT` config parameter, and ensure Kubernetes memory limits are set correspondingly higher to prevent OOM kills. Monitor cache hit rates and actual usage to find the optimal value for your filesystem size and access patterns.
