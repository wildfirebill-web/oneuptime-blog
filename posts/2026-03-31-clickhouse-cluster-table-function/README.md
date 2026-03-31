# How to Use cluster() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Table Function, Cluster, Distributed, SQL, Database

Description: Learn how to use the cluster() table function in ClickHouse to query all shards of a named cluster simultaneously, enabling cross-shard analytics and distributed data exploration.

---

The `cluster()` table function in ClickHouse lets you query a local table on every shard in a named cluster simultaneously. Instead of connecting to each shard individually, you issue one query from any node and ClickHouse fans it out across the entire cluster and merges the results.

## What Is the cluster() Table Function?

In a sharded ClickHouse deployment, each shard holds a subset of the data. The `cluster()` function wraps a local table reference and executes the query on all shards defined in the cluster configuration, returning a unified result set.

```sql
-- Query the 'logs' table on every shard in the 'production' cluster
SELECT count()
FROM cluster('production', default.logs);
```

## Basic Syntax

```sql
cluster(cluster_name, db.table)
cluster(cluster_name, db, table)
cluster(cluster_name, db.table, sharding_key)
```

| Parameter      | Description |
|----------------|-------------|
| `cluster_name` | Name of a cluster defined in `remote_servers` config |
| `db.table`     | The local table to query on each shard |
| `sharding_key` | Optional expression used for consistent shard targeting (rarely needed for reads) |

## Cluster Configuration

Before using `cluster()`, you need a cluster defined in your ClickHouse `config.xml` or an included file:

```xml
<remote_servers>
    <production>
        <shard>
            <replica>
                <host>ch-shard1-replica1.internal</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-shard1-replica2.internal</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>ch-shard2-replica1.internal</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-shard2-replica2.internal</host>
                <port>9000</port>
            </replica>
        </shard>
    </production>
</remote_servers>
```

## Querying All Shards

```sql
-- Count total events across all shards
SELECT count()
FROM cluster('production', default.events);

-- Aggregate metrics from all shards
SELECT
    toStartOfHour(ts) AS hour,
    sum(request_count) AS total_requests,
    avg(latency_ms)    AS avg_latency
FROM cluster('production', default.http_metrics)
WHERE ts >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;
```

## Inspecting Cluster Topology

Use the `system.clusters` table to see all configured clusters and their shards:

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_local
FROM system.clusters
WHERE cluster = 'production'
ORDER BY shard_num, replica_num;
```

## Using cluster() with Specific Databases

```sql
-- Query a table in a non-default database on all shards
SELECT
    user_id,
    count() AS session_count
FROM cluster('production', analytics, user_sessions)
WHERE session_start >= today()
GROUP BY user_id
ORDER BY session_count DESC
LIMIT 20;
```

## Comparing cluster() vs Distributed Tables

Both approaches query all shards, but they differ in setup and use:

| Feature | `cluster()` | Distributed table |
|---|---|---|
| Requires DDL | No | Yes (CREATE TABLE ... ENGINE=Distributed) |
| One-off queries | Great | Overkill |
| Repeated queries | Verbose | Preferred |
| Access control | Query-level | Table-level |

```sql
-- Distributed table approach (requires prior setup)
SELECT count() FROM dist_events;

-- cluster() approach (no prior setup)
SELECT count() FROM cluster('production', default.events);
```

## Joining Across Shards

You can join the result of `cluster()` with a local or another distributed source:

```sql
SELECT
    c.user_id,
    c.total_orders,
    u.username,
    u.email
FROM (
    SELECT user_id, count() AS total_orders
    FROM cluster('production', default.orders)
    GROUP BY user_id
) AS c
JOIN users AS u ON c.user_id = u.user_id
WHERE c.total_orders > 10;
```

## Checking Data Distribution Across Shards

Use `cluster()` to diagnose uneven data distribution:

```sql
SELECT
    _shard_num,
    count() AS row_count,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size
FROM cluster('production', system.parts)
WHERE table = 'events' AND active
GROUP BY _shard_num
ORDER BY _shard_num;
```

The `_shard_num` virtual column tells you which shard each row came from.

## Running DDL on All Shards

For DDL (CREATE, ALTER, DROP), use `ON CLUSTER` instead of `cluster()`:

```sql
-- Create a table on all shards simultaneously
CREATE TABLE default.events ON CLUSTER 'production'
(
    ts       DateTime,
    user_id  UInt64,
    event    String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, user_id);
```

## clusterAllReplicas() - Query Every Replica

The `clusterAllReplicas()` variant queries every replica (not just one per shard), useful for verifying replication consistency:

```sql
-- Check that all replicas have the same row count
SELECT
    hostName()  AS replica,
    count()     AS row_count
FROM clusterAllReplicas('production', default.events)
GROUP BY replica
ORDER BY replica;
```

## Security Considerations

- The query is sent to remote shards using the credentials of the current user. Ensure that user exists on all shards with the necessary permissions.
- Use `interserver_http_credentials` or TLS to secure cross-shard communication.
- `cluster()` bypasses row-level security if applied only on the querying node - apply policies on each shard individually.

## Performance Tips

- Add `WHERE` clauses that align with the partition key to prune data on each shard before network transfer.
- Use `LIMIT` carefully: it is applied after results are gathered from all shards, so each shard may return more rows than the final result.
- For heavy aggregations, the two-phase aggregation (`distributed_aggregation_memory_efficient`) setting reduces memory on the coordinator.

```sql
SET distributed_aggregation_memory_efficient = 1;

SELECT
    toDate(ts) AS date,
    count()    AS events
FROM cluster('production', default.events)
GROUP BY date
ORDER BY date;
```

## Summary

The `cluster()` table function provides ad-hoc distributed query capability without requiring pre-created Distributed tables. Key points:

- Query all shards of a named cluster with a single SQL statement.
- Use `_shard_num` to identify which shard each row originated from.
- Use `clusterAllReplicas()` when you need to query every replica for consistency checks.
- Prefer Distributed tables for frequently-run production queries; use `cluster()` for exploration and one-off operations.
