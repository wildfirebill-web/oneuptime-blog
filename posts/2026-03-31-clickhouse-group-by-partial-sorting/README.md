# How to Optimize ClickHouse GROUP BY with Partial Sorting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GROUP BY, Partial Sorting, Query Optimization, Aggregation

Description: Learn how ClickHouse's partial sorting optimization for GROUP BY reduces memory usage and speeds up aggregation when data is sorted by the grouping key.

---

ClickHouse can exploit pre-sorted data to optimize GROUP BY operations, reducing memory consumption and improving speed by avoiding hash table-based aggregation for sorted inputs.

## Standard GROUP BY Behavior

By default, ClickHouse uses a hash aggregation approach:

```sql
SELECT user_id, count() AS events
FROM events
GROUP BY user_id;
```

This builds a hash table of all unique `user_id` values in memory before producing output.

## Sorted Aggregation

When data is sorted by the GROUP BY key (e.g., `user_id` is part of the ORDER BY in MergeTree), ClickHouse can use a streaming aggregation that processes groups one at a time:

```sql
-- Table ordered by (user_id, ts)
-- This query can use sorted aggregation
SELECT user_id, count()
FROM events
GROUP BY user_id
ORDER BY user_id;
```

Enable the optimization explicitly:

```sql
SET optimize_aggregation_in_order = 1;
```

## Verifying Sorted Aggregation is Active

```sql
EXPLAIN PIPELINE
SELECT user_id, count()
FROM events
GROUP BY user_id
SETTINGS optimize_aggregation_in_order = 1;
```

Look for `AggregatingInOrderTransform` in the output, which indicates sorted aggregation is being used instead of standard hash aggregation.

## Memory Benefits

Standard hash aggregation keeps all keys in memory simultaneously. Sorted aggregation only keeps the current group in memory, making it viable for high-cardinality keys:

```sql
-- This might OOM with hash aggregation on billions of unique user_ids
-- With sorted aggregation, memory is bounded
SELECT user_id, sum(value), avg(value), count()
FROM events
GROUP BY user_id
SETTINGS optimize_aggregation_in_order = 1;
```

## Partial Sorting for Multiple Keys

For compound GROUP BY matching the sort order:

```sql
-- Table ORDER BY (project_id, user_id, ts)
-- Both keys benefit from partial sorting
SELECT project_id, user_id, count()
FROM events
GROUP BY project_id, user_id
ORDER BY project_id, user_id
SETTINGS optimize_aggregation_in_order = 1;
```

## Read Groups Before GROUP BY

For sorted data, you can also use `GROUP BY` with `LIMIT BY` to get the top N per group efficiently:

```sql
SELECT user_id, event_type, count()
FROM events
GROUP BY user_id, event_type
ORDER BY user_id, count() DESC
LIMIT 3 BY user_id;
```

## When Sorted Aggregation Does Not Help

- GROUP BY keys differ from sort order
- Filters change the effective sort order
- Data is heavily scattered across many parts

## Summary

ClickHouse's `optimize_aggregation_in_order` setting enables sorted aggregation, trading hash table memory for a streaming approach when data is pre-sorted by the GROUP BY key. This is especially valuable for high-cardinality aggregations on large MergeTree tables.
