# How to Use setup_instruments Table in MySQL Performance Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Instrument, Configuration, Monitoring

Description: Learn how to configure MySQL Performance Schema instrumentation using the setup_instruments table to control what events are collected and measured.

---

## Overview

The `setup_instruments` table is the master configuration table for MySQL Performance Schema. It controls which internal server operations are instrumented - from SQL statements and stored procedures to file I/O, memory allocations, and mutex operations. Understanding this table is essential for customizing Performance Schema to focus on what matters to you.

## Exploring Available Instruments

```sql
-- Count instruments by category
SELECT
  SUBSTRING_INDEX(NAME, '/', 2) AS category,
  COUNT(*) AS count,
  SUM(ENABLED = 'YES') AS enabled_count
FROM performance_schema.setup_instruments
GROUP BY category
ORDER BY count DESC;
```

## Viewing All Statement Instruments

```sql
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%'
ORDER BY NAME;
```

## Enabling and Disabling Instruments

To enable a specific category:

```sql
-- Enable all wait/io instruments with timing
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/io/%';

-- Disable memory instrumentation to reduce overhead
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME LIKE 'memory/%';
```

## Understanding ENABLED vs. TIMED

- `ENABLED = 'YES'` - events are collected but may not include timing
- `TIMED = 'YES'` - event timing is measured (adds small CPU overhead)

Collecting events without timing has minimal overhead. Enable timing only for instruments you actively need to profile.

## Enabling Specific High-Value Instruments

For a typical performance investigation:

```sql
-- Statements with full timing
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/sql/%';

-- Table I/O
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/io/table/%';

-- Lock waits
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/lock/%';
```

## Making Changes Persistent

By default, `setup_instruments` changes are lost on server restart. To persist them, use the `performance_schema_instrument` startup option in `my.cnf`:

```text
[mysqld]
performance_schema_instrument='statement/sql/%=ON'
performance_schema_instrument='wait/io/table/%=ON'
performance_schema_instrument='wait/lock/%=ON'
```

## Checking Current Instrument Status

```sql
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE ENABLED = 'YES'
  AND TIMED = 'YES'
  AND NAME LIKE 'wait/%'
ORDER BY NAME
LIMIT 20;
```

## Properties Column

```sql
SELECT NAME, ENABLED, TIMED, PROPERTIES, VOLATILITY
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/sql/select'
\G
```

The `PROPERTIES` column describes instrument characteristics, and `VOLATILITY` indicates how often instrument state changes (singleton vs. session-level).

## Summary

The `setup_instruments` table is the control panel for MySQL Performance Schema data collection. By selectively enabling instruments with timing for the specific wait, statement, or I/O categories you want to analyze, you can maximize diagnostic value while minimizing the performance overhead of instrumentation itself.
