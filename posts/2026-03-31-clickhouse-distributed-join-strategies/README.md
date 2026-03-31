# How to Handle Distributed JOIN Strategies in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed, JOIN, Strategy, Performance, Shard

Description: Explore the different strategies ClickHouse offers for executing JOINs across shards and how to pick the right one for your workload.

---

JOINs across Distributed tables are one of the trickier areas of ClickHouse. The default behavior, subquery broadcast, works for small dimension tables but falls apart at scale. Knowing which strategy to apply prevents full-table shuffles and keeps latency predictable.

## The Default: Broadcast the Right Side

By default ClickHouse rewrites a distributed JOIN by sending the right-hand side as a subquery to every shard. Each shard then performs a local join.

```sql
SELECT o.user_id, u.name, o.amount
FROM dist_orders AS o
JOIN dist_users AS u ON o.user_id = u.user_id
LIMIT 100;
```

If `dist_users` is large this transfers the entire right table to every shard - expensive and memory-intensive.

## distributed_product_mode

The `distributed_product_mode` setting controls how nested subqueries referencing Distributed tables are handled:

```sql
SET distributed_product_mode = 'local';
-- options: deny, local, global, allow
```

- `deny` - raises an error if a distributed table appears in a subquery.
- `local` - replaces the distributed subquery with the local underlying table on each shard.
- `global` - broadcasts the subquery result to all shards (equivalent to GLOBAL IN).
- `allow` - permits full cross-shard subqueries (may produce duplicate rows).

## Collocated Joins

The most efficient strategy is to shard both tables on the join key. Each shard then joins rows locally with no network transfer.

```sql
-- Both tables sharded by user_id
SELECT o.user_id, u.name, count()
FROM dist_orders  AS o
JOIN dist_users   AS u ON o.user_id = u.user_id
GROUP BY o.user_id, u.name;
```

With collocated sharding, the query plan shows only local reads.

## GLOBAL JOIN for Non-Collocated Tables

When sharding keys differ, use GLOBAL JOIN to broadcast the smaller table once:

```sql
SELECT o.user_id, u.name, sum(o.amount)
FROM dist_orders AS o
GLOBAL JOIN dist_users AS u ON o.user_id = u.user_id
GROUP BY o.user_id, u.name;
```

The initiator executes `SELECT * FROM dist_users`, loads the result into memory, and sends it to every shard as a temporary table. Shards perform local lookups against it.

## Choosing a Strategy

| Scenario | Strategy |
|---|---|
| Right table fits in memory | GLOBAL JOIN |
| Both tables sharded on join key | Collocated (default) |
| Right table is huge | Rethink schema or use a dictionary |
| Ad-hoc analytics | GLOBAL JOIN with LIMIT on right side |

## Summary

ClickHouse distributed JOINs default to broadcasting the right side, which is efficient only for small dimension tables. Collocated sharding eliminates network transfer entirely for equality joins. For mismatched sharding keys, GLOBAL JOIN broadcasts the smaller relation once and keeps shard-side computation local.
