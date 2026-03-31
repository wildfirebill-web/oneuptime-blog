# How to Handle Duplicate Events in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Deduplication, Duplicate Events, ReplacingMergeTree, Data Quality

Description: Learn the key strategies for detecting and removing duplicate events in ClickHouse using table engines, query-time deduplication, and background merges.

---

## Sources of Duplicate Events

Duplicates enter ClickHouse from producer retries, at-least-once Kafka consumers, double-processing during pipeline restarts, and manual reloads. Each strategy for handling them has different tradeoffs between storage, query complexity, and latency.

## Query-Time Deduplication with DISTINCT

The simplest approach is to deduplicate at query time using `DISTINCT` or `GROUP BY`:

```sql
SELECT DISTINCT event_id, user_id, event_time, value
FROM events
WHERE event_time >= today() - 1;
```

For aggregations, use `uniq` instead of `count`:

```sql
SELECT user_id, uniq(event_id) AS unique_events
FROM events
WHERE event_time >= today()
GROUP BY user_id;
```

## ReplacingMergeTree

Use `ReplacingMergeTree` to keep only the latest version of each row based on a sort key:

```sql
CREATE TABLE events
(
    event_id    String,
    event_time  DateTime,
    user_id     UInt64,
    value       Float64,
    ingested_at DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(ingested_at)
ORDER BY event_id;
```

After background merges, only the row with the latest `ingested_at` survives per `event_id`. Use `FINAL` to get consistent reads immediately:

```sql
SELECT event_id, value
FROM events FINAL
WHERE event_time >= today();
```

## CollapsingMergeTree

For explicit cancel-and-replace semantics, use `CollapsingMergeTree`:

```sql
CREATE TABLE events
(
    event_id    String,
    user_id     UInt64,
    value       Float64,
    sign        Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY event_id;

-- Insert original
INSERT INTO events VALUES ('evt-1', 42, 9.99, 1);

-- Cancel original and insert corrected
INSERT INTO events VALUES ('evt-1', 42, 9.99, -1), ('evt-1', 42, 12.99, 1);
```

## Detecting Duplicates

Find duplicates before removing them:

```sql
SELECT event_id, count() AS cnt
FROM events
GROUP BY event_id
HAVING cnt > 1
ORDER BY cnt DESC
LIMIT 100;
```

## Scheduled Deduplication

For non-replicated tables, run periodic deduplication:

```sql
OPTIMIZE TABLE events DEDUPLICATE BY event_id;
```

This collapses exact duplicate rows. Use it with caution on large tables as it triggers a full merge.

## Summary

Handle duplicate events in ClickHouse by choosing the right table engine - `ReplacingMergeTree` for version-based deduplication, `CollapsingMergeTree` for explicit cancellations - and using `FINAL` for query-time consistency. For simpler cases, `OPTIMIZE DEDUPLICATE` or `DISTINCT` in queries may be sufficient.
