# How to Use GLOBAL JOIN in ClickHouse Distributed Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SELECT, JOIN, GLOBAL JOIN, Distributed

Description: Learn how GLOBAL JOIN in ClickHouse broadcasts the right-side table to all shards, preventing incorrect results and reducing network overhead in distributed queries.

---

When running a JOIN against a distributed table in ClickHouse, the default behavior sends the right-side subquery to every shard, where each shard executes it independently. This can produce incorrect results if the right-side data is itself distributed, because each shard only sees its local slice. The `GLOBAL` keyword changes this behavior: ClickHouse executes the right-side query once on the initiator node, then broadcasts the complete result to all shards before the join is performed locally on each shard.

## The Problem with Regular Distributed JOINs

Without `GLOBAL`, the right-side subquery runs on each shard independently, leading to each shard only joining against its local partition of the right table:

```sql
-- Potentially incorrect: each shard runs the subquery against its local data only
SELECT
    e.event_id,
    e.user_id,
    u.name
FROM distributed_events AS e
INNER JOIN distributed_users AS u ON e.user_id = u.user_id;
-- A shard may not find the user if the user lives on a different shard
```

## GLOBAL JOIN Syntax

Add `GLOBAL` between the join type and `JOIN` keyword:

```sql
-- Correct: right-side query runs once on initiator, result broadcast to all shards
SELECT
    e.event_id,
    e.user_id,
    u.name
FROM distributed_events AS e
GLOBAL INNER JOIN distributed_users AS u ON e.user_id = u.user_id;
```

## GLOBAL with Other Join Types

`GLOBAL` works with all join types:

```sql
-- GLOBAL LEFT JOIN
SELECT
    u.user_id,
    u.name,
    count(e.event_id) AS event_count
FROM distributed_users AS u
GLOBAL LEFT JOIN distributed_events AS e ON u.user_id = e.user_id
GROUP BY u.user_id, u.name;

-- GLOBAL ANY LEFT JOIN
SELECT
    e.event_id,
    e.event_time,
    u.country
FROM distributed_events AS e
GLOBAL ANY LEFT JOIN distributed_users AS u ON e.user_id = u.user_id;

-- GLOBAL SEMI JOIN
SELECT e.event_id, e.event_time
FROM distributed_events AS e
GLOBAL LEFT SEMI JOIN (
    SELECT DISTINCT user_id FROM vip_users
) AS vip ON e.user_id = vip.user_id;
```

## How GLOBAL JOIN Works Internally

```text
1. Initiator node receives the query.
2. The right-side subquery (or table) is executed on the initiator:
      SELECT * FROM distributed_users
   This collects all matching rows from all shards into a temporary table.
3. The temporary table is serialized and broadcast to every shard.
4. Each shard executes the left-side query against its local data
   and joins with the received temporary table.
5. Results are merged back at the initiator.
```

This means the right-side data is transferred across the network once per query, not once per shard-to-shard lookup.

## GLOBAL JOIN vs Regular JOIN - When to Use Each

```sql
-- Use GLOBAL JOIN when:
-- 1. The right-side table is distributed and small enough to broadcast
-- 2. You need correct results across all shards

-- Use regular JOIN when:
-- 1. The right-side table is a local (non-distributed) table on each node
-- 2. The right-side table is a small static lookup loaded locally
-- 3. You are querying a single shard directly

-- Example: local dimension table does NOT need GLOBAL
SELECT e.event_id, c.country_name
FROM distributed_events AS e
INNER JOIN local_country_lookup AS c ON e.country_code = c.country_code;
-- local_country_lookup exists identically on every node, no broadcast needed
```

## Memory and Network Considerations

The right-side result is held in memory on the initiator and then sent to all shards. If the right-side table is large, this can cause memory pressure on the initiator and significant network traffic.

```sql
-- Control broadcast table size with max_bytes_in_join
SET max_bytes_in_join = 1073741824;  -- 1 GB limit

-- For very large right-side tables, consider pre-filtering before the GLOBAL JOIN
SELECT
    e.event_id,
    u.name
FROM distributed_events AS e
GLOBAL INNER JOIN (
    SELECT user_id, name
    FROM distributed_users
    WHERE country = 'US'         -- filter first to reduce broadcast size
) AS u ON e.user_id = u.user_id;
```

## GLOBAL IN vs GLOBAL JOIN

ClickHouse also supports `GLOBAL IN` for the same broadcast semantics with subqueries in `WHERE` clauses:

```sql
-- GLOBAL IN: broadcast the subquery result to all shards for the IN filter
SELECT event_id, event_time
FROM distributed_events
WHERE user_id GLOBAL IN (
    SELECT user_id FROM distributed_users WHERE tier = 'premium'
);

-- Equivalent GLOBAL SEMI JOIN form
SELECT e.event_id, e.event_time
FROM distributed_events AS e
GLOBAL LEFT SEMI JOIN (
    SELECT user_id FROM distributed_users WHERE tier = 'premium'
) AS p ON e.user_id = p.user_id;
```

## Summary

`GLOBAL JOIN` ensures correct results in distributed ClickHouse queries by executing the right-side subquery once on the initiator node and broadcasting the complete result to all shards before local joins are performed. Without `GLOBAL`, each shard runs the right-side subquery independently against its local data, which produces incomplete matches when both tables are distributed across multiple shards. Use `GLOBAL JOIN` when the right-side table is distributed and small enough to broadcast; avoid it for very large right-side datasets that would overwhelm memory or network bandwidth.
