# How to Configure RGW for Read-Heavy Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Object Storage, Performance

Description: Configure Ceph RGW in Rook for read-heavy workloads by enabling object caching, tuning readahead, and deploying cache tiers to maximize GET throughput.

---

## Read-Heavy Workload Profile

Read-heavy patterns include content delivery, ML model serving, static asset hosting, and analytics query results caching. These workloads generate many GET requests for the same objects repeatedly. Caching and readahead are the primary optimization levers.

## Enabling the RGW Object Cache

Enable the in-memory metadata and small-object cache:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_cache_enabled true

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_cache_lru_size 500000

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_cache_expiry_interval 3600
```

Increase the maximum cacheable object size:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_cache_fetch_max_obj_size 10485760
```

## RADOS Readahead

For large object GETs, increase the readahead buffer:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_get_obj_max_req_size 33554432

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client client_readahead_max_bytes 67108864
```

## BlueStore Read Cache

Maximize BlueStore's ability to serve reads from memory on OSD nodes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_size_ssd 8589934592

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_data_ratio 0.8

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_kv_ratio 0.1
```

## RGW Instances for Read Scaling

Scale out RGW instances to parallelize read requests:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: read-store
  namespace: rook-ceph
spec:
  gateway:
    instances: 4
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "8"
        memory: "16Gi"
```

## Thread Pool for Read Concurrency

Read requests are typically short-lived. Use a large thread pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_thread_pool_size 1024

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_num_rados_handles 16
```

## Range Request Optimization

For applications that use HTTP Range requests, tune the request window:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_range_request_cache_max_size 524288000
```

## Benchmarking Read Throughput

Test sustained GET throughput:

```bash
warp get \
  --host=<rgw-endpoint>:80 \
  --access-key=<key> \
  --secret-key=<secret> \
  --concurrent=128 \
  --obj.size=1MiB \
  --duration=2m
```

Monitor cache hit rate in Prometheus:

```text
rate(ceph_rgw_cache_hit[5m]) / rate(ceph_rgw_cache_miss[5m])
```

A ratio above 10 indicates effective caching.

## Summary

Read-heavy RGW optimization layers the built-in metadata cache with BlueStore data caching and RADOS readahead to maximize cache hit rates. Scaling RGW instances and increasing thread pool size allows more parallel reads without OSD contention. For workloads with highly predictable access patterns, a cache tier on SSD OSDs can absorb nearly all read I/O.
