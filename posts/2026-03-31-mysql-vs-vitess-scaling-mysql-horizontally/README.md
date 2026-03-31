# MySQL vs Vitess: Scaling MySQL Horizontally

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Vitess, Sharding

Description: Learn how Vitess extends MySQL with horizontal sharding, connection pooling, and automated failover, and when to add Vitess to your MySQL deployment.

---

Vitess is not a replacement for MySQL - it is a middleware layer that runs in front of MySQL to add horizontal sharding, connection pooling, and query routing. PlanetScale, YouTube, and Slack have all used Vitess to scale MySQL beyond single-node limits.

## What Vitess Adds to MySQL

A plain MySQL setup has one primary and optional replicas. All writes go to one server, which creates a ceiling.

Vitess introduces:
- **VTGate** - a stateless SQL router that clients connect to
- **VTTablet** - a sidecar process per MySQL instance
- **Topology server** - ZooKeeper or etcd stores shard metadata

```text
Application -> VTGate -> VTTablet (shard 1, MySQL primary)
                      -> VTTablet (shard 2, MySQL primary)
                      -> VTTablet (shard 3, MySQL primary)
```

## Connection Pooling

MySQL has a hard limit on connections (controlled by `max_connections`). Each idle connection consumes memory. VTTablet implements server-side connection pooling, multiplexing thousands of application connections onto a small pool of MySQL connections.

```sql
-- MySQL: check current connections
SHOW STATUS LIKE 'Threads_connected';

-- With Vitess: application sees thousands of connections
-- VTTablet maintains a pool of ~200 MySQL connections per shard
```

## Sharding

Without Vitess, sharding MySQL requires application-level logic to determine which database server holds each row.

With Vitess, you define a sharding key and Vitess routes queries automatically.

```sql
-- Vitess VSchema: define sharding key for the orders table
-- (defined in JSON VSchema, not SQL)
```

```json
{
  "sharded": true,
  "vindexes": {
    "hash": { "type": "hash" }
  },
  "tables": {
    "orders": {
      "column_vindexes": [
        { "column": "user_id", "name": "hash" }
      ]
    }
  }
}
```

Queries with the sharding key in the WHERE clause are routed to the correct shard. Scatter queries (without the key) fan out to all shards.

## Query Rewriting and Validation

VTGate rewrites queries before forwarding them to MySQL. It blocks full-table scans on large tables, enforces query timeouts, and normalizes queries for caching.

```sql
-- Vitess rejects queries without a WHERE clause on sharded tables
-- to prevent accidental scatter queries
SELECT * FROM orders; -- blocked by Vitess policy
SELECT * FROM orders WHERE user_id = 42; -- routed to correct shard
```

## Schema Changes

Vitess includes an online schema change tool (Online DDL) that applies schema migrations without table locks, using gh-ost or pt-osc under the hood.

```sql
-- Vitess Online DDL
ALTER TABLE orders ADD COLUMN notes TEXT,
ALGORITHM=ONLINE, LOCK=NONE;
-- Vitess routes this through gh-ost automatically
```

## When to Add Vitess

| Situation | Recommendation |
|---|---|
| Single MySQL node, < 1TB data | Plain MySQL, no Vitess needed |
| Need connection pooling only | ProxySQL is simpler |
| Write throughput exceeds single-node | Consider Vitess |
| Need horizontal sharding | Vitess is the MySQL-native answer |
| Using PlanetScale | Vitess is built-in |

## Operational Complexity

Vitess adds significant operational complexity - VTGate, VTTablet, topology server, and VTAdmin all need to be deployed and managed. The learning curve is steep. PlanetScale provides Vitess as a managed service.

## Summary

Vitess extends MySQL with horizontal sharding, connection pooling, and automated failover without changing your SQL dialect. It is the right choice when a single MySQL node is no longer sufficient for writes and you want to stay on MySQL. For simpler scaling needs, read replicas and ProxySQL cover most cases without Vitess's operational overhead.
