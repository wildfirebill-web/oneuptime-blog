# How to Use the events_stages_summary Tables in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Stage, Query Execution, Diagnostic

Description: Use MySQL Performance Schema events_stages_summary tables to understand query execution stages and identify which phases consume the most time.

---

## Overview

MySQL executes queries in multiple stages: parsing, opening tables, sorting, locking, sending data, and more. The `events_stages_summary_*` tables aggregate time spent in each stage, enabling you to pinpoint precisely which phase is causing slowdowns.

## Available Stage Summary Tables

- `events_stages_summary_global_by_event_name` - global totals
- `events_stages_summary_by_thread_by_event_name` - per thread
- `events_stages_summary_by_account_by_event_name` - per account
- `events_stages_summary_by_user_by_event_name` - per user
- `events_stages_summary_by_host_by_event_name` - per host

## Enabling Stage Instrumentation

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'stage/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE '%stages%';
```

## Finding the Most Time-Consuming Query Stages

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR AS stage_count,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms
FROM performance_schema.events_stages_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 15;
```

## Identifying Sorting and Temp Table Issues

High time in sorting or temp table stages signals missing indexes or improper query design:

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.events_stages_summary_global_by_event_name
WHERE EVENT_NAME IN (
  'stage/sql/Sorting result',
  'stage/sql/Creating sort index',
  'stage/sql/Creating tmp table',
  'stage/sql/Copying to tmp table'
)
ORDER BY SUM_TIMER_WAIT DESC;
```

## Locking Stage Analysis

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(MAX_TIMER_WAIT / 1e9, 2) AS max_ms
FROM performance_schema.events_stages_summary_global_by_event_name
WHERE EVENT_NAME LIKE '%lock%'
   OR EVENT_NAME LIKE '%waiting%'
ORDER BY SUM_TIMER_WAIT DESC;
```

## Per-Thread Stage Breakdown

Find threads spending the most time in expensive stages:

```sql
SELECT
  t.PROCESSLIST_USER,
  s.EVENT_NAME,
  s.COUNT_STAR,
  ROUND(s.SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.events_stages_summary_by_thread_by_event_name s
JOIN performance_schema.threads t USING (THREAD_ID)
WHERE s.COUNT_STAR > 0
  AND t.TYPE = 'FOREGROUND'
ORDER BY s.SUM_TIMER_WAIT DESC
LIMIT 20;
```

## Monitoring Alter Table Progress

For long-running DDL, you can watch stage progress in `events_stages_current`:

```sql
SELECT
  EVENT_NAME,
  WORK_COMPLETED,
  WORK_ESTIMATED,
  ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 1) AS pct_complete
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%';
```

## Summary

The `events_stages_summary_global_by_event_name` table provides a stage-by-stage breakdown of query execution time. By focusing on sorting, temp table creation, and lock waiting stages, you can diagnose issues that are not visible at the query level and make targeted improvements to indexes, query structure, and server configuration.
