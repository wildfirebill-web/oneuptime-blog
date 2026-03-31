# How to Use groupBitmap() Aggregate Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitmap, Aggregate Function, Analytics, Materialized View

Description: Learn how groupBitmap() and its State/Merge variants build and combine Roaring Bitmaps in ClickHouse aggregations, powering fast unique-user counts and segment analytics.

---

`groupBitmap(column)` is an aggregate function that collects all `UInt64` values in a group into a single Roaring Bitmap and returns the cardinality (distinct count). Its companion `groupBitmapState(column)` produces the intermediate bitmap state as `AggregateFunction(groupBitmap, UInt64)`, which can be stored in `AggregatingMergeTree` tables. `groupBitmapMerge(state_column)` merges stored states back into a final cardinality. Together they form the standard pattern for pre-aggregated unique-ID analytics in ClickHouse.

## Basic groupBitmap() Usage

```sql
-- Count distinct users per event type in one pass
SELECT
    event_type,
    groupBitmap(toUInt64(user_id)) AS unique_users
FROM events
GROUP BY event_type
ORDER BY unique_users DESC;
```

This is equivalent to `uniqExact(user_id)` but the state can be persisted for later merging.

## groupBitmapState() in a Materialized View

```sql
-- Raw events table
CREATE TABLE events
(
    event_time  DateTime,
    event_date  Date DEFAULT toDate(event_time),
    event_type  LowCardinality(String),
    user_id     UInt64
)
ENGINE = MergeTree()
ORDER BY (event_type, event_date, user_id);
```

```sql
-- Aggregating table for daily per-event-type bitmaps
CREATE TABLE events_daily_bitmap
(
    event_date  Date,
    event_type  LowCardinality(String),
    user_bitmap AggregateFunction(groupBitmap, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (event_type, event_date);
```

```sql
-- Materialized view that populates the bitmap table on insert
CREATE MATERIALIZED VIEW events_daily_bitmap_mv
TO events_daily_bitmap
AS
SELECT
    event_date,
    event_type,
    groupBitmapState(user_id) AS user_bitmap
FROM events
GROUP BY event_date, event_type;
```

## groupBitmapMerge() - Reading Back the State

```sql
-- Query unique users per event type for a date range
SELECT
    event_type,
    groupBitmapMerge(user_bitmap) AS unique_users
FROM events_daily_bitmap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY event_type
ORDER BY unique_users DESC;
```

`groupBitmapMerge` correctly de-duplicates user IDs that appear on multiple days.

## Inserting Test Data

```sql
INSERT INTO events (event_time, event_type, user_id)
SELECT
    now() - (rand() % 86400),
    ['page_view', 'click', 'purchase'][1 + (rand() % 3)],
    rand() % 100000
FROM numbers(1000000);
```

## Merging States for Multi-Day Unique Counts

```sql
-- Unique users who performed EACH event type in June (correct deduplication across days)
SELECT
    event_type,
    groupBitmapMerge(user_bitmap) AS june_unique_users
FROM events_daily_bitmap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY event_type;
```

```sql
-- Total unique users across ALL event types in June (union across types and days)
SELECT
    bitmapCardinality(groupBitmapOrState(user_bitmap)) AS total_unique_users_june
FROM events_daily_bitmap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-30';
```

## groupBitmapOrState and groupBitmapAndState

When you want to merge multiple bitmap rows into a single result bitmap (rather than a count), use the state aggregators.

```sql
-- Union bitmap: all users who appeared on any day in June
SELECT
    event_type,
    bitmapCardinality(groupBitmapOrState(user_bitmap)) AS total_unique,
    bitmapToArray(groupBitmapOrState(user_bitmap))     AS all_user_ids  -- careful with large bitmaps
FROM events_daily_bitmap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07'
GROUP BY event_type;
```

```sql
-- Intersection bitmap: users who appeared on EVERY day in a 7-day window
SELECT
    event_type,
    bitmapCardinality(groupBitmapAndState(user_bitmap)) AS daily_active_all_7_days
FROM events_daily_bitmap
WHERE event_date BETWEEN '2024-06-01' AND '2024-06-07'
GROUP BY event_type;
```

## Cross-Segment Unique Count Using Stored Bitmaps

```sql
-- How many users made at least one page_view AND at least one purchase in June?
WITH
    (
        SELECT groupBitmapOrState(user_bitmap)
        FROM events_daily_bitmap
        WHERE event_type = 'page_view'
          AND event_date BETWEEN '2024-06-01' AND '2024-06-30'
    ) AS viewers,
    (
        SELECT groupBitmapOrState(user_bitmap)
        FROM events_daily_bitmap
        WHERE event_type = 'purchase'
          AND event_date BETWEEN '2024-06-01' AND '2024-06-30'
    ) AS purchasers
SELECT
    bitmapAndCardinality(viewers, purchasers) AS viewed_and_purchased;
```

## Partial Aggregation with -State and -Merge

The State/Merge pattern enables two-level aggregation for distributed queries.

```sql
-- First level: produce per-shard partial states
SELECT
    toStartOfWeek(event_date) AS week,
    event_type,
    groupBitmapState(user_id) AS partial_bitmap
FROM events
GROUP BY week, event_type;
```

```sql
-- Second level: merge partial states for final counts
SELECT
    week,
    event_type,
    groupBitmapMerge(partial_bitmap) AS weekly_unique_users
FROM weekly_partial_states
GROUP BY week, event_type
ORDER BY week, weekly_unique_users DESC;
```

## Comparing groupBitmap vs uniqExact

```sql
-- Both return correct distinct counts; groupBitmap state is mergeable
SELECT
    event_type,
    uniqExact(user_id)             AS uniq_exact,
    groupBitmap(toUInt64(user_id)) AS bitmap_count
FROM events
GROUP BY event_type;
-- Results are identical; bitmap approach supports incremental merging
```

## Summary

`groupBitmap(col)` aggregates a column of `UInt64` IDs into a Roaring Bitmap and returns its cardinality. `groupBitmapState(col)` produces a storable `AggregateFunction(groupBitmap, UInt64)` intermediate state, and `groupBitmapMerge(state_col)` merges multiple stored states back into a final count with correct deduplication. Use `groupBitmapOrState` to union bitmaps across rows and `groupBitmapAndState` to intersect them. The State/Merge pattern stored in `AggregatingMergeTree` is the standard architecture for fast, pre-aggregated unique-user analytics in ClickHouse at scale.
