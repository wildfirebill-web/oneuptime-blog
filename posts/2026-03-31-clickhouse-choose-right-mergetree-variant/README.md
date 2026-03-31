# How to Choose the Right MergeTree Variant in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MergeTree, ReplacingMergeTree, SummingMergeTree, AggregatingMergeTree, Engine, Schema Design

Description: Learn how to choose the right MergeTree engine variant in ClickHouse based on your data mutability, aggregation, deduplication, and time-series requirements.

---

ClickHouse offers seven MergeTree engine variants, each designed for a specific data pattern. Choosing the wrong engine leads to duplicate data, wasted storage, or unnecessary query complexity. This guide walks through each variant and its ideal use case.

## The Seven MergeTree Variants

```text
Engine                       Key Capability
MergeTree                    Base engine, append-only
ReplacingMergeTree           Deduplication by key + version
SummingMergeTree             Pre-aggregated numeric sums
AggregatingMergeTree         Generic pre-aggregation (any aggregate state)
CollapsingMergeTree          Sign-based row cancellation
VersionedCollapsingMergeTree Sign-based collapse with version ordering
GraphiteMergeTree            Graphite time-series rollup
```

## Use MergeTree When: Append-Only Facts

Pure event logs, telemetry, and immutable fact tables. No deduplication needed.

```sql
CREATE TABLE events (
    ts       DateTime,
    user_id  UInt64,
    event    String
) ENGINE = MergeTree PARTITION BY toYYYYMM(ts) ORDER BY (ts, user_id);
```

## Use ReplacingMergeTree When: Mutable Records (CDC)

Use when each record has a stable natural key and you need the latest version - user profiles, order statuses, product inventory.

```sql
ENGINE = ReplacingMergeTree(version)
ORDER BY user_id
```

Query with `FINAL` to force deduplication at read time.

## Use SummingMergeTree When: Pre-Summing Metrics

Use when you aggregate the same numeric columns repeatedly by the same GROUP BY key. Merges pre-sum the values, so queries are faster.

```sql
CREATE TABLE daily_revenue (
    day        Date,
    product_id UInt32,
    revenue    Float64,
    order_count UInt64
) ENGINE = SummingMergeTree((revenue, order_count))
PARTITION BY toYYYYMM(day)
ORDER BY (day, product_id);
```

## Use AggregatingMergeTree When: Complex Pre-Aggregation

When you need more than sums - e.g., uniq counts, quantiles, averages - combine AggregatingMergeTree with a Materialized View and `AggregateFunction` column types.

```sql
ENGINE = AggregatingMergeTree()
ORDER BY (day, product_id)
```

This is the backing engine for MV-based pre-aggregation pipelines.

## Use CollapsingMergeTree When: State Corrections via Sign

Use for mutable state where you correct a previous row by inserting its sign-inverse. Common in financial systems.

```sql
ENGINE = CollapsingMergeTree(sign)
ORDER BY order_id
```

Insert `sign=1` for new state, `sign=-1` to cancel a previous state.

## Use VersionedCollapsingMergeTree When: Unordered State Updates

Like CollapsingMergeTree but handles out-of-order sign rows correctly using a version column.

```sql
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY order_id
```

## Use GraphiteMergeTree When: Graphite Metrics Rollup

Specifically for storing Graphite-format time-series data with automatic rollup rules.

```sql
ENGINE = GraphiteMergeTree('graphite_rollup')
ORDER BY (metric, timestamp)
```

## Decision Tree

```text
Need mutable records (upserts)?
  Yes, with numeric sums only -> SummingMergeTree
  Yes, with complex aggregates -> AggregatingMergeTree (via MV)
  Yes, keep latest row by key -> ReplacingMergeTree
  Yes, sign-based corrections (ordered) -> CollapsingMergeTree
  Yes, sign-based corrections (unordered) -> VersionedCollapsingMergeTree
No mutations, append-only? -> MergeTree
Graphite metrics rollup? -> GraphiteMergeTree
```

## Summary

Start with `MergeTree` for append-only facts, use `ReplacingMergeTree` for CDC-style mutable records, `SummingMergeTree` for pre-summing counters, and `AggregatingMergeTree` for arbitrary pre-aggregation behind materialized views. Choose `VersionedCollapsingMergeTree` over `CollapsingMergeTree` when event ordering cannot be guaranteed.
