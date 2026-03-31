# How to Optimize Distributed GROUP BY in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed, GROUP BY, Aggregation, Performance, Shard

Description: Practical techniques for speeding up GROUP BY queries on Distributed tables by reducing data shuffled to the initiator node.

---

GROUP BY on a Distributed table forces partial aggregates from every shard to be sent to the initiator, which then performs the final merge. When groups are large or cardinality is high, this merge step becomes the bottleneck. Several ClickHouse features directly address this.

## Push Down the Aggregation to Shards

If the GROUP BY key matches your sharding key, ClickHouse can fully aggregate on each shard and send only summary rows to the initiator.

```sql
-- Distributed table sharded by user_id
SELECT user_id, sum(revenue)
FROM dist_orders
GROUP BY user_id;
```

When `user_id` is the sharding key, each shard owns all rows for a given `user_id`, so the shard result is already final. The initiator only concatenates rows.

## Use distributed_group_by_no_merge

When you know the aggregation is already complete on shards, instruct the initiator not to re-merge:

```sql
SET distributed_group_by_no_merge = 1;

SELECT user_id, sum(revenue)
FROM dist_orders
GROUP BY user_id;
```

Use this only when sharding guarantees complete groups per shard, otherwise results will be incorrect.

## Two-Level Aggregation

Enable two-level aggregation to reduce memory pressure during the merge phase:

```sql
SET group_by_two_level_threshold = 50000;
SET group_by_two_level_threshold_bytes = 20000000;
```

With two-level aggregation, shards partition their hash table into 256 buckets. The initiator merges one bucket at a time, keeping peak memory bounded.

## Limit Data Transferred with HAVING Push-Down

Move HAVING filters into a subquery so shards discard low-volume groups early:

```sql
SELECT user_id, total
FROM (
    SELECT user_id, sum(revenue) AS total
    FROM dist_orders
    GROUP BY user_id
) WHERE total > 1000;
```

ClickHouse pushes the HAVING predicate into the shard sub-query, meaning small groups never travel over the network.

## Monitor Aggregation Memory

```sql
SELECT
    query_id,
    formatReadableSize(memory_usage) AS mem,
    ProfileEvents['AggregationPreallocatedElementsInHashTables'] AS prealloced
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%dist_orders%'
ORDER BY memory_usage DESC
LIMIT 5;
```

## Summary

Optimize distributed GROUP BY by aligning the GROUP BY key with the sharding key, enabling two-level aggregation, and pushing HAVING filters into sub-queries. When sharding guarantees group completeness, `distributed_group_by_no_merge = 1` eliminates the initiator merge entirely for maximum throughput.
