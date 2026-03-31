# How to Fix "Maximum distributed connections reached" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed, Connection Pool, Troubleshooting, Performance

Description: Resolve ClickHouse "Maximum distributed connections reached" errors by tuning connection pool limits and reducing concurrent distributed query load.

---

## Understanding the Error

When ClickHouse's distributed query engine exhausts its connection pool to remote shards, queries fail with:

```text
DB::Exception: Too many connections to replica. Maximum connections: 1024. (TOO_MANY_SIMULTANEOUS_QUERIES)
```

or:

```text
DB::Exception: Maximum distributed connections reached. (NETWORK_ERROR)
```

This happens under heavy concurrent distributed query load when the connection pool to remote shards fills up.

## Checking the Current Connection Pool Status

```sql
-- Check active connections and query concurrency
SELECT metric, value
FROM system.metrics
WHERE metric LIKE '%Connection%' OR metric LIKE '%Distributed%';

-- See active distributed queries
SELECT
    query_id,
    user,
    elapsed,
    left(query, 200) AS q
FROM system.processes
WHERE query LIKE '%Distributed%' OR query LIKE '%_distributed%'
ORDER BY elapsed DESC;
```

## Fix 1 - Increase the Distributed Connection Pool Size

In `/etc/clickhouse-server/config.xml`:

```xml
<!-- Maximum number of connections in distributed query connection pool -->
<distributed_connections_pool_size>1024</distributed_connections_pool_size>

<!-- Connections per endpoint -->
<max_connections>4096</max_connections>
```

Or at the session level:

```sql
SET distributed_connections_pool_size = 2048;
```

## Fix 2 - Reduce Distributed Query Concurrency

Limit how many distributed queries can run simultaneously per user:

```xml
<!-- users.xml -->
<profiles>
  <analysts>
    <max_concurrent_queries_for_user>20</max_concurrent_queries_for_user>
    <max_concurrent_select_queries_for_user>15</max_concurrent_select_queries_for_user>
  </analysts>
</profiles>
```

Server-wide:

```xml
<!-- config.xml -->
<max_concurrent_queries>100</max_concurrent_queries>
```

## Fix 3 - Enable Connection Pooling with Compression

Reduce the number of required connections by enabling compression, which lets fewer connections carry more data:

```xml
<!-- config.xml -->
<compression>
  <case>
    <method>lz4</method>
  </case>
</compression>
```

## Fix 4 - Use Connection Reuse (Keep-Alive)

Ensure ClickHouse reuses TCP connections between distributed queries:

```xml
<keep_alive_timeout>10</keep_alive_timeout>
```

## Fix 5 - Queue Distributed Queries

Instead of failing, queue excess queries:

```xml
<!-- config.xml -->
<concurrent_threads_soft_limit>0</concurrent_threads_soft_limit>
<!-- Enable query queue -->
<max_waiting_queries>100</max_waiting_queries>
```

## Identifying the Bottleneck

```sql
-- Find which shards are receiving the most connections
SELECT
    host_name,
    host_port,
    errors_count,
    estimated_recovery_time
FROM system.clusters
ORDER BY errors_count DESC;

-- Check if specific queries are monopolizing connections
SELECT
    user,
    count() AS query_count,
    max(elapsed) AS max_elapsed_sec
FROM system.processes
GROUP BY user
ORDER BY query_count DESC;
```

## Using Async Distributed Inserts

For write-heavy workloads, use async distributed inserts to reduce connection pressure:

```sql
-- Buffer inserts asynchronously to avoid connection spikes
SET async_insert = 1;
SET wait_for_async_insert = 0;
SET async_insert_max_data_size = 10000000;
SET async_insert_busy_timeout_ms = 200;

INSERT INTO analytics.events_distributed
SELECT * FROM analytics.staging_events;
```

## Summary

"Maximum distributed connections reached" errors occur when many concurrent distributed queries exhaust the connection pool to remote shards. Increase `distributed_connections_pool_size` in `config.xml` for immediate relief, and limit per-user concurrency with `max_concurrent_queries_for_user`. For sustained high-throughput scenarios, combine connection reuse via keep-alive, async inserts, and query queuing to balance load without hitting hard connection limits.
