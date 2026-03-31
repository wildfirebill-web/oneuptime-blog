# How to Use the events_waits_summary Tables in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Wait Event, Latency, Diagnostic

Description: Learn how to use MySQL Performance Schema events_waits_summary tables to diagnose lock contention, I/O waits, and concurrency bottlenecks.

---

## Overview

Wait events are the foundation of MySQL performance analysis. Whenever MySQL has to wait for a lock, an I/O operation, a mutex, or any other resource, it records a wait event. The `events_waits_summary_*` tables aggregate these waits to show you exactly where time is being lost.

## Available Wait Summary Tables

- `events_waits_summary_global_by_event_name` - global totals per event type
- `events_waits_summary_by_thread_by_event_name` - per thread breakdown
- `events_waits_summary_by_account_by_event_name` - per account
- `events_waits_summary_by_user_by_event_name` - per user
- `events_waits_summary_by_host_by_event_name` - per host

## Enabling Wait Instrumentation

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE '%waits%';
```

## Finding the Most Time-Consuming Wait Events Globally

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR AS wait_count,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
  ROUND(MAX_TIMER_WAIT / 1e9, 2) AS max_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 15;
```

## Identifying Lock Wait Hotspots

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(AVG_TIMER_WAIT / 1e6, 2) AS avg_us
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/synch/mutex/%'
   OR EVENT_NAME LIKE 'wait/lock/%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Per-Thread Wait Analysis

Find which threads are spending the most time waiting:

```sql
SELECT
  t.PROCESSLIST_USER,
  t.PROCESSLIST_HOST,
  w.EVENT_NAME,
  w.COUNT_STAR,
  ROUND(w.SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.events_waits_summary_by_thread_by_event_name w
JOIN performance_schema.threads t USING (THREAD_ID)
WHERE w.COUNT_STAR > 0
  AND t.TYPE = 'FOREGROUND'
ORDER BY w.SUM_TIMER_WAIT DESC
LIMIT 15;
```

## I/O vs. Lock vs. Sync Wait Breakdown

```sql
SELECT
  CASE
    WHEN EVENT_NAME LIKE 'wait/io/%' THEN 'I/O'
    WHEN EVENT_NAME LIKE 'wait/lock/%' THEN 'Lock'
    WHEN EVENT_NAME LIKE 'wait/synch/%' THEN 'Sync'
    ELSE 'Other'
  END AS category,
  SUM(COUNT_STAR) AS total_waits,
  ROUND(SUM(SUM_TIMER_WAIT) / 1e12, 2) AS total_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
GROUP BY category
ORDER BY total_sec DESC;
```

## Resetting Wait Statistics

```sql
TRUNCATE TABLE performance_schema.events_waits_summary_global_by_event_name;
```

## Summary

The `events_waits_summary_global_by_event_name` table is the starting point for diagnosing MySQL performance at a low level. By categorizing waits into I/O, lock, and synchronization types, you can quickly determine whether your bottleneck is disk-bound, concurrency-bound, or lock-contention-bound, and apply the appropriate remedy.
