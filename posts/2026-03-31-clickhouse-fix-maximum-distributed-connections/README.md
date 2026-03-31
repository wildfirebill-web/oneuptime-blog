# How to Fix "Maximum distributed connections reached" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed, Connection Pool, Error, Troubleshooting

Description: Resolve "Maximum distributed connections reached" errors in ClickHouse by tuning connection pool limits and optimizing distributed query concurrency.

---

ClickHouse enforces a maximum number of simultaneous connections to remote shards during distributed query execution. When this limit is hit, you see: `DB::Exception: Maximum distributed connections count reached`. This typically occurs under high query concurrency.

## Check Current Connection Limits

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'max_distributed_connections',
    'max_connections',
    'distributed_connections_pool_size'
);
```

## Increase the Distributed Connection Limit

For a single query:

```sql
SET max_distributed_connections = 1024;
SELECT count() FROM distributed_table WHERE event_date = today();
```

Persistently in `users.xml`:

```xml
<profiles>
  <default>
    <max_distributed_connections>1024</max_distributed_connections>
  </default>
</profiles>
```

## Increase the Connection Pool Size

In `config.xml`, increase the global connection pool:

```xml
<max_connections>4096</max_connections>
```

## Reduce Query Concurrency

If too many distributed queries run simultaneously, use a query queue or rate limit:

```xml
<max_concurrent_queries>100</max_concurrent_queries>
<max_concurrent_queries_for_user>20</max_concurrent_queries_for_user>
```

## Use Hedged Requests

Hedged requests reduce the number of connections needed by cancelling slow replicas:

```sql
SET use_hedged_requests = 1;
SET hedged_connection_timeout_ms = 100;
```

## Check Shard Connection Overhead

Each distributed query opens connections to all shards simultaneously. For a 20-shard cluster with 50 concurrent queries, that is 1000 connections. Optimize by:

1. Routing queries to specific shards using `shardNum()` filtering
2. Using async inserts to batch writes
3. Reducing the number of concurrent distributed SELECTs

```sql
-- Limit to a specific shard range
SELECT count() FROM distributed_table
WHERE cityHash64(user_id) % 20 = 0;  -- Only hits shard 0
```

## Monitor Active Connections

```sql
SELECT
    count() AS active_queries,
    sum(distributed_connections_used) AS dist_connections
FROM system.processes;
```

## Summary

"Maximum distributed connections reached" occurs when too many simultaneous distributed queries exhaust the connection pool. Raise `max_distributed_connections` per query or globally, increase `max_connections` in `config.xml`, and control concurrency with `max_concurrent_queries` limits. Long-term, reduce the number of shards queried per request by routing queries to specific shards.
