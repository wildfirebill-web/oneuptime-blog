# How to Use POPULATE vs Manual Backfill for Materialized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, POPULATE, Backfill, Data Migration

Description: Learn the differences between POPULATE and manual INSERT SELECT backfill for ClickHouse materialized views, and when to use each approach.

---

## Two Ways to Backfill

When you create a materialized view on an existing table, new data is processed automatically but historical data is not. ClickHouse offers two approaches: the `POPULATE` keyword and manual INSERT SELECT.

## The POPULATE Keyword

`POPULATE` creates the view and immediately runs a backfill query:

```sql
CREATE MATERIALIZED VIEW mv_events_hourly
TO events_hourly
POPULATE
AS SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
GROUP BY hour, event_type;
```

### How POPULATE Works

1. ClickHouse runs `SELECT ... FROM source GROUP BY ...` on all existing data.
2. Results are inserted into the target table.
3. The view is then activated for future inserts.

### POPULATE Risks

The critical problem: any inserts to the source table that happen between step 1 and step 3 are silently lost from the target. This is documented ClickHouse behavior.

```text
-- Timeline with POPULATE risk:
T=0: CREATE MATERIALIZED VIEW ... POPULATE
T=1: POPULATE backfill reads data from source
T=2: New rows inserted to source (LOST - not captured yet)
T=3: POPULATE completes, view activates
T=4: New rows now flow to target normally
```

## When to Use POPULATE

Use POPULATE only when:

- The source table has no active writes during view creation
- You are working in a development or staging environment
- The data loss window is acceptable

```sql
-- Safe to use in dev with no active inserts
CREATE MATERIALIZED VIEW mv_dev_events
TO dev_events
POPULATE
AS SELECT * FROM raw_dev_events;
```

## Manual Backfill with INSERT SELECT

The safe production approach: create the view without POPULATE, then backfill manually.

```sql
-- Step 1: Create view WITHOUT populate (no historical data yet)
CREATE MATERIALIZED VIEW mv_events_hourly
TO events_hourly
AS SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
GROUP BY hour, event_type;

-- Step 2: Backfill historical data manually
INSERT INTO events_hourly
SELECT
    toStartOfHour(event_time) AS hour,
    event_type,
    count() AS cnt
FROM raw_events
WHERE event_time < now()
GROUP BY hour, event_type;
```

This approach has zero data loss because the view starts capturing live data before the backfill runs, and the backfill covers history up to the view creation point.

## Backfill in Chunks for Large Tables

```sql
-- Backfill one month at a time
INSERT INTO events_hourly
SELECT toStartOfHour(event_time), event_type, count()
FROM raw_events
WHERE toYYYYMM(event_time) = 202501
GROUP BY 1, 2;
```

## Verify Completeness

```sql
SELECT
    toYYYYMM(hour) AS month,
    sum(cnt) AS backfilled_events
FROM events_hourly
GROUP BY month
ORDER BY month;
```

## Summary

Prefer manual INSERT SELECT over `POPULATE` for production tables with active writes. POPULATE has a data loss window that is unacceptable in most production scenarios. Manual backfill with chunked inserts provides full control, zero data loss, and verifiable completeness.
