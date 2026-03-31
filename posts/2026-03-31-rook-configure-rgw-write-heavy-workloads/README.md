# How to Configure RGW for Write-Heavy Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Object Storage, Performance

Description: Configure Ceph RGW in Rook for write-heavy workloads by tuning write buffers, OSD journals, bucket index performance, and scaling RGW instances for ingestion pipelines.

---

## Write-Heavy Workload Profile

Write-heavy patterns include log aggregation, IoT sensor data ingestion, backup pipelines, and media upload systems. These workloads generate a constant stream of PUT requests and require RGW to efficiently flush data to OSDs without creating backpressure.

## Increasing Write Buffer and Chunk Size

Configure RGW to buffer larger writes before flushing to RADOS:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_max_chunk_size 33554432

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_put_obj_max_window_size 67108864
```

## OSD Journal and BlueStore Write Tuning

Increase BlueStore write ahead log (WAL) and DB capacity to absorb write bursts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_write_ahead_log_max_size 1073741824

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd bluestore_cache_size_ssd 4294967296
```

Configure OSD to prefer writes over reads when queue fills:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_op_queue mclock_scheduler

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set osd osd_mclock_scheduler_client_res 100
```

## Bucket Index Optimization for Writes

High-rate writes to a single bucket create bucket index contention. Pre-shard the index:

```bash
# New buckets
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_override_bucket_index_max_shards 128

# Existing buckets receiving heavy writes
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket reshard --bucket=log-bucket --num-shards=128
```

## RGW Instance Scaling

Add more RGW instances to increase write throughput:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: ingest-store
  namespace: rook-ceph
spec:
  gateway:
    instances: 6
    resources:
      requests:
        cpu: "4"
        memory: "8Gi"
      limits:
        cpu: "8"
        memory: "16Gi"
```

Distribute writers across all RGW instances using round-robin DNS or a load balancer.

## Async Logging for Write Latency

Disable synchronous request logging to reduce write latency:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_ops_log_rados false

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_enable_ops_log false
```

For high-volume logging, use a dedicated ops log pool instead of the default inline logging.

## Benchmarking Write Throughput

Test sustained PUT throughput with warp:

```bash
warp put \
  --host=<rgw-endpoint>:80 \
  --access-key=<key> \
  --secret-key=<secret> \
  --concurrent=64 \
  --obj.size=1MiB \
  --duration=2m
```

Monitor OSD write latency:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd perf | sort -k4 -n | tail -10
```

## Summary

Write-heavy RGW workloads benefit from larger write buffers at the RGW level, pre-sharded bucket indexes to distribute index writes, and BlueStore tuning to absorb write spikes. Disabling synchronous ops logging removes a common hidden bottleneck for high-rate write workloads.
