# How to Query INFORMATION_SCHEMA.EVENTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Event, Scheduler, Metadata

Description: Learn how to query INFORMATION_SCHEMA.EVENTS in MySQL to list scheduled events, inspect execution schedules, and monitor event status and history.

---

## Overview

MySQL Event Scheduler allows you to run scheduled SQL jobs automatically. `INFORMATION_SCHEMA.EVENTS` provides metadata about all scheduled events, including their schedule type, next execution time, status, and SQL body. This is the programmatic alternative to `SHOW EVENTS`.

## Basic Query

```sql
SELECT
  EVENT_SCHEMA,
  EVENT_NAME,
  EVENT_TYPE,
  EXECUTE_AT,
  INTERVAL_VALUE,
  INTERVAL_FIELD,
  STATUS,
  LAST_EXECUTED,
  STARTS,
  ENDS,
  DEFINER
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
ORDER BY EVENT_NAME;
```

## Key Columns

| Column | Description |
|--------|-------------|
| `EVENT_NAME` | Name of the scheduled event |
| `EVENT_TYPE` | ONE TIME or RECURRING |
| `EXECUTE_AT` | Execution time for ONE TIME events |
| `INTERVAL_VALUE` | Numeric interval for RECURRING events |
| `INTERVAL_FIELD` | Unit for interval (MINUTE, HOUR, DAY, etc.) |
| `STATUS` | ENABLED, DISABLED, or SLAVESIDE_DISABLED |
| `LAST_EXECUTED` | Timestamp of last execution |
| `STARTS` / `ENDS` | Validity window for recurring events |
| `EVENT_DEFINITION` | SQL body of the event |
| `DEFINER` | user@host of creator |

## Listing All Enabled Events

```sql
SELECT
  EVENT_NAME,
  EVENT_TYPE,
  INTERVAL_VALUE,
  INTERVAL_FIELD,
  LAST_EXECUTED,
  STATUS
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
  AND STATUS = 'ENABLED'
ORDER BY EVENT_NAME;
```

## Finding One-Time Events Not Yet Executed

```sql
SELECT
  EVENT_NAME,
  EXECUTE_AT,
  TIMEDIFF(EXECUTE_AT, NOW()) AS time_until_execution
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
  AND EVENT_TYPE = 'ONE TIME'
  AND STATUS = 'ENABLED'
  AND EXECUTE_AT > NOW()
ORDER BY EXECUTE_AT;
```

## Finding Events That Have Never Run

```sql
SELECT EVENT_NAME, CREATED, STATUS
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
  AND LAST_EXECUTED IS NULL;
```

## Reading an Event Definition

```sql
SELECT EVENT_DEFINITION
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
  AND EVENT_NAME = 'cleanup_old_sessions'\G
```

## Finding Events Near Their End Date

```sql
SELECT
  EVENT_NAME,
  ENDS,
  DATEDIFF(ENDS, NOW()) AS days_until_expiry
FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'myapp'
  AND ENDS IS NOT NULL
  AND ENDS > NOW()
  AND DATEDIFF(ENDS, NOW()) < 30
ORDER BY ENDS;
```

## Enabling the Event Scheduler

Before events can run, the scheduler must be enabled:

```sql
SHOW VARIABLES LIKE 'event_scheduler';
SET GLOBAL event_scheduler = ON;
```

## Summary

`INFORMATION_SCHEMA.EVENTS` provides a complete catalog of all scheduled MySQL events. By querying it, you can monitor execution schedules, detect disabled or expired events, review event definitions, and ensure your scheduled maintenance jobs (log cleanup, statistics refresh, partition management) are running correctly.
