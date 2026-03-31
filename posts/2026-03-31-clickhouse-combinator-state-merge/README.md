# How to Use -State and -Merge Combinators in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Aggregate Function, Combinator, AggregatingMergeTree, Materialized View, Performance

Description: Learn how -State saves intermediate aggregate binary state and -Merge combines those states, enabling incremental aggregation with AggregatingMergeTree and materialized views.

---

Most ClickHouse aggregate functions compute a final answer directly. But in high-throughput systems you often want to pre-aggregate data incrementally - storing partial results that can later be merged without re-reading raw rows. That is the role of the `-State` and `-Merge` combinators. `-State` instructs an aggregate function to return its internal binary state instead of a final value. `-Merge` takes those stored states and combines them into a final result. Together they are the engine behind `AggregatingMergeTree` tables and materialized views that aggregate in the background as data arrives.

## How -State Works

Appending `-State` to any aggregate function name changes its return type from the normal result type to `AggregateFunction(funcName, argTypes...)`. This binary blob encodes everything the function needs to continue aggregating later.

```text
aggFuncState(column)  ->  AggregateFunction(aggFunc, column_type)
```

The binary value is opaque - you cannot inspect it directly. You must use `-Merge` or `-MergeState` to read it.

## How -Merge Works

Appending `-Merge` combines multiple stored `AggregateFunction` states into a single final result.

```text
aggFuncMerge(state_column)  ->  final result type
```

You use `-Merge` in SELECT queries against tables that store `-State` values. You use `-MergeState` when you want to combine states from multiple rows and store the result as yet another state (useful when aggregating over materialized views further upstream).

## A Practical Materialized View Pipeline

The canonical use case is a materialized view backed by `AggregatingMergeTree`. Raw events arrive in a source table. A materialized view pre-aggregates them using `-State` and stores the results. A SELECT query at read time uses `-Merge` to finalize the aggregation.

### Step 1: Create the Raw Events Table

```sql
CREATE TABLE raw_events
(
    event_time  DateTime,
    user_id     UInt32,
    page        String,
    duration_ms UInt32,
    error       UInt8   -- 1 if the request errored
)
ENGINE = MergeTree()
ORDER BY (event_time, user_id);
```

### Step 2: Create the Aggregating Table

Use `AggregatingMergeTree`. Columns that store states must have type `AggregateFunction(...)`.

```sql
CREATE TABLE page_stats_agg
(
    page             String,
    hour             DateTime,
    visit_count      AggregateFunction(count),
    total_duration   AggregateFunction(sum, UInt32),
    avg_duration     AggregateFunction(avg, UInt32),
    error_count      AggregateFunction(sum, UInt8),
    uniq_users       AggregateFunction(uniq, UInt32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (page, hour);
```

### Step 3: Create the Materialized View

The view uses `-State` combinators to produce the right column types.

```sql
CREATE MATERIALIZED VIEW page_stats_mv
TO page_stats_agg
AS
SELECT
    page,
    toStartOfHour(event_time)    AS hour,
    countState()                 AS visit_count,
    sumState(duration_ms)        AS total_duration,
    avgState(duration_ms)        AS avg_duration,
    sumState(error)              AS error_count,
    uniqState(user_id)           AS uniq_users
FROM raw_events
GROUP BY page, hour;
```

### Step 4: Insert Raw Events

Data inserted into `raw_events` is automatically processed by the materialized view and written to `page_stats_agg`.

```sql
INSERT INTO raw_events VALUES
    ('2026-03-31 10:05:00', 1, '/home',    250, 0),
    ('2026-03-31 10:12:00', 2, '/home',    380, 0),
    ('2026-03-31 10:20:00', 1, '/about',   120, 0),
    ('2026-03-31 10:45:00', 3, '/home',    900, 1),
    ('2026-03-31 11:02:00', 2, '/home',    200, 0),
    ('2026-03-31 11:15:00', 4, '/about',   450, 1);
```

### Step 5: Query with -Merge

At read time, use `-Merge` to finalize each state column into its actual value.

```sql
SELECT
    page,
    hour,
    countMerge(visit_count)     AS visits,
    sumMerge(total_duration)    AS total_ms,
    avgMerge(avg_duration)      AS avg_ms,
    sumMerge(error_count)       AS errors,
    uniqMerge(uniq_users)       AS unique_users
FROM page_stats_agg
GROUP BY page, hour
ORDER BY page, hour;
```

```text
page    hour                 visits  total_ms  avg_ms  errors  unique_users
/about  2026-03-31 10:00:00  1       120       120     0       1
/about  2026-03-31 11:00:00  1       450       450     1       1
/home   2026-03-31 10:00:00  3       1530      510     1       3
/home   2026-03-31 11:00:00  1       200       200     0       1
```

Note the `GROUP BY` in the final SELECT. `AggregatingMergeTree` merges parts over time, but a query may read multiple unmerged parts. The `GROUP BY` with `-Merge` handles any remaining unmerged states.

## Manual -State / -Merge Without a Materialized View

You can also use `-State` and `-Merge` manually, for example to store a snapshot of aggregated state in a regular table.

```sql
-- Store a daily snapshot of aggregate state
CREATE TABLE daily_snapshots
(
    snapshot_date  Date,
    avg_state      AggregateFunction(avg, Float64)
)
ENGINE = MergeTree()
ORDER BY snapshot_date;

INSERT INTO daily_snapshots
SELECT
    today()          AS snapshot_date,
    avgState(amount) AS avg_state
FROM orders;

-- Read back the final average
SELECT avgMerge(avg_state) AS overall_avg
FROM daily_snapshots
WHERE snapshot_date = today();
```

## Combining -MergeState for Multi-Level Aggregation

When you need to merge states from two different materialized views (e.g., hourly into daily), use `-MergeState` to produce a new merged state that can be stored again.

```sql
-- Merge hourly states into a daily summary state
INSERT INTO daily_page_stats
SELECT
    page,
    toStartOfDay(hour)          AS day,
    countMergeState(visit_count)    AS visit_count,
    avgMergeState(avg_duration)     AS avg_duration
FROM page_stats_agg
GROUP BY page, day;
```

This pattern lets you build multi-tier aggregation hierarchies (raw -> hourly -> daily -> monthly) without ever re-reading the original raw events.

## Summary

The `-State` combinator changes an aggregate function to return a binary intermediate state instead of a final value, stored as `AggregateFunction(funcName, ...)`. The `-Merge` combinator finalizes those stored states back into the actual result. Together they power `AggregatingMergeTree` tables and materialized views, enabling incremental pre-aggregation that scales with write throughput. Always remember to apply `GROUP BY` with `-Merge` at read time to handle unmerged parts, and use `-MergeState` when you need to combine states from one tier into another in a multi-level aggregation pipeline.
