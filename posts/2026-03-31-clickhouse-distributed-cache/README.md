# How to Use Distributed Cache in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed Cache, Performance, Caching, Cluster

Description: Learn how ClickHouse's distributed cache allows multiple nodes in a cluster to share a single cached dataset, reducing redundant reads from object storage.

---

In a ClickHouse cluster backed by shared object storage, every node independently caches the same remote data by default. The distributed cache feature changes this by routing cache requests to a small set of dedicated cache nodes, so identical data is cached only once and shared across all query nodes.

## Why Use Distributed Cache

Without distributed caching, a cluster of 10 nodes might each cache the same hot partition, wasting local disk space and still hitting object storage during cache misses on a given node. With distributed caching, one cache node holds the data and all other nodes fetch it locally over the fast internal network rather than from S3.

## Configuring the Cache Coordinator

Add a `distributed_cache` section to your ClickHouse configuration:

```xml
<clickhouse>
  <filesystem_caches>
    <s3_shared_cache>
      <path>/var/lib/clickhouse/filesystem_cache/shared/</path>
      <max_size>214748364800</max_size> <!-- 200 GB -->
      <cache_on_write_operations>1</cache_on_write_operations>
    </s3_shared_cache>
  </filesystem_caches>

  <distributed_cache>
    <enabled>true</enabled>
    <pool_size>16</pool_size>
    <queue_wait_max_milliseconds>1000</queue_wait_max_milliseconds>
  </distributed_cache>
</clickhouse>
```

The `pool_size` controls the number of background threads on each node used to serve cache requests from peers.

## Storage Disk with Distributed Cache

Reference the cache name from your disk configuration:

```xml
<s3_disk>
  <type>s3</type>
  <endpoint>https://s3.amazonaws.com/mybucket/</endpoint>
  <access_key_id>ACCESS_KEY</access_key_id>
  <secret_access_key>SECRET_KEY</secret_access_key>
  <cache_name>s3_shared_cache</cache_name>
  <use_distributed_cache>true</use_distributed_cache>
</s3_disk>
```

## Verifying Distributed Cache Activity

Check the system events table to confirm distributed cache reads and writes are occurring:

```sql
SELECT event, value
FROM system.events
WHERE event LIKE '%DistributedCache%'
ORDER BY event;
```

Key events to watch:
- `DistributedCacheReadMicroseconds` - latency for cache fetches from a peer
- `DistributedCacheHit` - successful peer cache reads
- `DistributedCacheMiss` - cache misses that required going to object storage

## Checking Cache Distribution Across Nodes

```sql
SELECT
    hostname() AS node,
    cache_name,
    formatReadableSize(sum(size)) AS cached_bytes
FROM clusterAllReplicas('my_cluster', system.filesystem_cache)
GROUP BY node, cache_name
ORDER BY node;
```

This lets you verify that cache data is being distributed rather than replicated on every node.

## Tuning Distributed Cache Behavior

```sql
-- Allow this query to use distributed cache
SET enable_filesystem_cache = 1;

-- Set timeout for waiting on a peer cache node
SET distributed_cache_wait_connection_timeout_milliseconds = 500;

-- Fall back to S3 if peer cache is unavailable
SET distributed_cache_throw_on_error = 0;
```

Setting `distributed_cache_throw_on_error = 0` ensures queries remain functional even if a cache node becomes temporarily unreachable.

## When to Use Distributed Cache

Distributed caching is most beneficial when:
- You have many query replicas sharing object storage
- Your hot data is larger than what any single node can cache locally
- Cache miss latency to object storage is a bottleneck

For small clusters or when local SSD capacity is plentiful, per-node filesystem caching may be simpler and sufficient.

## Summary

ClickHouse's distributed cache eliminates redundant object-storage reads across cluster nodes by sharing a unified cache layer. Configure the `use_distributed_cache` flag on your S3 disk, tune the pool size to match your concurrency, and monitor `DistributedCacheHit` events to confirm the cache is working effectively across your cluster.
