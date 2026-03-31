# How to Use AggregatingMergeTree in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AggregatingMergeTree, Aggregation, Database, Engine, SQL, Performance

Description: Learn how to use the AggregatingMergeTree engine in ClickHouse to store partial aggregation states for complex functions like uniq, quantile, and avg, enabling efficient incremental pre-aggregation.

---

AggregatingMergeTree is the most powerful pre-aggregation engine in ClickHouse. Unlike SummingMergeTree which only sums numbers, AggregatingMergeTree can store and merge the intermediate "states" of any aggregate function - including `uniq`, `avg`, `quantile`, `groupArray`, and custom combinators.

## What Is AggregatingMergeTree?

During background merges, AggregatingMergeTree merges the stored aggregate states of rows that share the same `ORDER BY` key. The aggregate states are binary-encoded partial results that can be combined to produce a final value.

The key concept: instead of storing the final result, you store an intermediate state using `*State` aggregate function combinators, and finalize with `*Merge` combinators at query time.

```sql
-- State combinator: stores intermediate result
avgState(value) -> AggregateFunction(avg, Float64)

-- Merge combinator: combines states and returns the final result
avgMerge(avg_state_column) -> Float64
```

## Basic Setup: Create the Table

```sql
CREATE TABLE user_daily_stats
(
    date         Date,
    user_id      UInt64,
    -- Store states, not final values
    session_count     AggregateFunction(count),
    total_revenue     AggregateFunction(sum, Float64),
    unique_pages      AggregateFunction(uniq, String),
    avg_session_ms    AggregateFunction(avg, UInt32),
    p95_latency       AggregateFunction(quantile(0.95), UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, user_id);
```

## Inserting State Data

Use `*State` combinators to create states when inserting:

```sql
INSERT INTO user_daily_stats
SELECT
    toDate(ts)            AS date,
    user_id,
    countState()          AS session_count,
    sumState(revenue)     AS total_revenue,
    uniqState(page_url)   AS unique_pages,
    avgState(duration_ms) AS avg_session_ms,
    quantileState(0.95)(duration_ms) AS p95_latency
FROM raw_sessions
GROUP BY date, user_id;
```

## Querying: Finalize with *Merge Combinators

```sql
SELECT
    date,
    user_id,
    countMerge(session_count)          AS sessions,
    sumMerge(total_revenue)            AS revenue,
    uniqMerge(unique_pages)            AS pages_visited,
    avgMerge(avg_session_ms)           AS avg_duration_ms,
    quantileMerge(0.95)(p95_latency)   AS p95_ms
FROM user_daily_stats
GROUP BY date, user_id
ORDER BY date DESC, revenue DESC
LIMIT 50;
```

## The Canonical Pattern: Materialized View

In production, AggregatingMergeTree tables are almost always populated by a materialized view:

```sql
-- 1. Raw events table
CREATE TABLE events
(
    ts         DateTime,
    user_id    UInt64,
    page_url   String,
    duration_ms UInt32,
    revenue    Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, user_id);

-- 2. Pre-aggregated state table
CREATE TABLE events_daily_agg
(
    date              Date,
    user_id           UInt64,
    hit_count         AggregateFunction(count),
    total_revenue     AggregateFunction(sum, Float64),
    unique_pages      AggregateFunction(uniq, String),
    avg_duration      AggregateFunction(avg, UInt32),
    p99_duration      AggregateFunction(quantile(0.99), UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, user_id);

-- 3. Materialized view that feeds the aggregated table
CREATE MATERIALIZED VIEW events_daily_agg_mv
TO events_daily_agg
AS
SELECT
    toDate(ts)                AS date,
    user_id,
    countState()              AS hit_count,
    sumState(revenue)         AS total_revenue,
    uniqState(page_url)       AS unique_pages,
    avgState(duration_ms)     AS avg_duration,
    quantileState(0.99)(duration_ms) AS p99_duration
FROM events
GROUP BY date, user_id;
```

Now every insert into `events` automatically updates `events_daily_agg`.

## Querying the Aggregated Table

```sql
-- Daily summary for the last 7 days
SELECT
    date,
    countMerge(hit_count)              AS total_hits,
    round(sumMerge(total_revenue), 2)  AS total_revenue,
    uniqMerge(unique_pages)            AS unique_pages,
    round(avgMerge(avg_duration))      AS avg_duration_ms,
    round(quantileMerge(0.99)(p99_duration)) AS p99_duration_ms
FROM events_daily_agg
WHERE date >= today() - 7
GROUP BY date
ORDER BY date DESC;
```

## Using -If Combinators for Conditional Aggregation

```sql
CREATE TABLE ab_test_stats
(
    date            Date,
    experiment_id   String,
    conversions_a   AggregateFunction(countIf, UInt8),
    conversions_b   AggregateFunction(countIf, UInt8),
    revenue_a       AggregateFunction(sumIf, Float64, UInt8),
    revenue_b       AggregateFunction(sumIf, Float64, UInt8)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, experiment_id);

-- Insert states using -If combinator
INSERT INTO ab_test_stats
SELECT
    toDate(ts)                   AS date,
    experiment_id,
    countIfState(variant = 'A')  AS conversions_a,
    countIfState(variant = 'B')  AS conversions_b,
    sumIfState(revenue, variant = 'A') AS revenue_a,
    sumIfState(revenue, variant = 'B') AS revenue_b
FROM experiment_events
GROUP BY date, experiment_id;

-- Query final results
SELECT
    date,
    experiment_id,
    countIfMerge(conversions_a) AS conv_a,
    countIfMerge(conversions_b) AS conv_b,
    round(sumIfMerge(revenue_a), 2) AS rev_a,
    round(sumIfMerge(revenue_b), 2) AS rev_b
FROM ab_test_stats
GROUP BY date, experiment_id
ORDER BY date DESC;
```

## Combining Multiple Granularities

Store both hourly and daily granularities in separate tables:

```sql
-- Hourly stats
CREATE TABLE events_hourly_agg
(
    hour      DateTime,
    user_id   UInt64,
    hit_count AggregateFunction(count),
    revenue   AggregateFunction(sum, Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (hour, user_id);

-- Daily stats
CREATE TABLE events_daily_agg2
(
    date      Date,
    user_id   UInt64,
    hit_count AggregateFunction(count),
    revenue   AggregateFunction(sum, Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, user_id);
```

## Using FINAL for Small Tables

For tables where query-time merge is acceptable:

```sql
SELECT
    date,
    uniqMerge(unique_pages) AS unique_pages
FROM events_daily_agg FINAL
WHERE date = today()
GROUP BY date;
```

## AggregateFunction Column Type Reference

```sql
-- Common aggregate state column types
hit_count     AggregateFunction(count)
visit_count   AggregateFunction(countIf, UInt8)
total_sum     AggregateFunction(sum, Float64)
average       AggregateFunction(avg, Float32)
approx_uniq   AggregateFunction(uniq, String)
approx_uniq64 AggregateFunction(uniq, UInt64)
median        AggregateFunction(median, Float64)
p95           AggregateFunction(quantile(0.95), UInt32)
top10         AggregateFunction(topK(10), String)
bitmap_agg    AggregateFunction(groupBitmap, UInt64)
```

## Performance Comparison

For a table with 1 billion events:

| Query | Engine | Time |
|---|---|---|
| `SELECT count(DISTINCT user_id) FROM events` | MergeTree | ~8 seconds |
| `SELECT uniqMerge(u) FROM events_daily_agg GROUP BY ...` | AggregatingMergeTree | ~80 ms |

The difference is orders of magnitude because AggregatingMergeTree works on pre-computed partial states.

## Summary

AggregatingMergeTree is the foundation of fast analytics dashboards in ClickHouse. Key points:

- Store partial aggregate states using `*State` combinators.
- Finalize states at query time using `*Merge` combinators.
- Always use a materialized view to populate AggregatingMergeTree tables automatically.
- Always `GROUP BY` the primary key columns in queries to handle partially merged state.
- Supports any aggregate function including `uniq`, `quantile`, `topK`, `groupBitmap`, and conditional `-If` variants.
