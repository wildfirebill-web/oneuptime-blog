# How to Use Replicated Database Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replicated Database Engine, DDL Replication, Cluster, High Availability

Description: Learn how to use the Replicated database engine in ClickHouse to automatically replicate DDL operations across all cluster nodes without ON CLUSTER.

---

The Replicated database engine provides automatic DDL replication across ClickHouse cluster nodes. When you create, alter, or drop a table in a Replicated database on one node, that change is propagated to all other nodes that share the same database. This simplifies cluster management by eliminating the need to run `ON CLUSTER` for every schema change.

## How It Differs from ON CLUSTER

Without Replicated database engine:

```sql
-- Must specify ON CLUSTER for every DDL operation
CREATE TABLE events ON CLUSTER my_cluster (...) ENGINE = ReplicatedMergeTree(...);
ALTER TABLE events ON CLUSTER my_cluster ADD COLUMN new_col String;
```

With Replicated database engine:

```sql
-- DDL automatically propagates to all nodes sharing this database
CREATE TABLE events (...) ENGINE = ReplicatedMergeTree(...);
ALTER TABLE events ADD COLUMN new_col String;
```

## Creating the Database

```sql
-- Run on each node that should share this database
CREATE DATABASE replicated_db
ENGINE = Replicated(
    '/clickhouse/databases/replicated_db',
    '{shard}',
    '{replica}'
);
```

- The first argument is the ZooKeeper path for coordination.
- `{shard}` and `{replica}` reference macros defined in each node's config.

## Creating Tables

Tables created inside a Replicated database should use ReplicatedMergeTree engines:

```sql
USE replicated_db;

CREATE TABLE events (
    event_id UInt64,
    event_type String,
    ts DateTime,
    user_id UInt32
)
ENGINE = ReplicatedMergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (event_type, ts);
```

Note: When using the Replicated database engine, you can omit the ZooKeeper path from the table engine - it is automatically assigned.

## DDL Propagation in Action

After creating the table on node 1, verify it appears on all nodes:

```sql
-- On node 2 (after propagation delay)
SELECT name, engine
FROM system.tables
WHERE database = 'replicated_db';
```

## Schema Changes

ALTER TABLE statements propagate automatically:

```sql
-- Run on any one node
ALTER TABLE replicated_db.events
    ADD COLUMN session_id String DEFAULT '',
    ADD COLUMN platform LowCardinality(String);
```

All nodes apply the change in order using the ZooKeeper coordination log.

## Monitoring DDL Replication

```sql
-- Check for pending DDL tasks
SELECT *
FROM system.distributed_ddl_queue
WHERE database = 'replicated_db'
ORDER BY entry_time DESC
LIMIT 20;
```

## Handling Node Failures During DDL

If a node is down when a DDL runs, the Replicated database engine will retry applying the DDL when the node reconnects. You can configure timeout behavior:

```xml
<!-- config.xml -->
<distributed_ddl>
    <task_max_lifetime>604800</task_max_lifetime>
    <cleanup_delay_period>60</cleanup_delay_period>
</distributed_ddl>
```

## Summary

The Replicated database engine eliminates the overhead of manually specifying `ON CLUSTER` for every schema change in a ClickHouse cluster. It uses ZooKeeper to coordinate DDL replication across all participating nodes, ensuring consistent schema state. Combined with ReplicatedMergeTree tables, it provides a fully automated, self-healing cluster schema management solution.
