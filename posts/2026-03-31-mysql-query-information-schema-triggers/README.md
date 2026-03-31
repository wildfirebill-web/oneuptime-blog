# How to Query INFORMATION_SCHEMA.TRIGGERS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Trigger, Schema, Metadata

Description: Learn how to query INFORMATION_SCHEMA.TRIGGERS in MySQL to list all triggers, inspect their definitions, timing, events, and execution order.

---

## Overview

`INFORMATION_SCHEMA.TRIGGERS` provides complete metadata about all triggers defined in MySQL. Triggers are often invisible in normal schema documentation, making this view essential for audits and understanding hidden business logic embedded in the database.

## Basic Query

```sql
SELECT
  TRIGGER_SCHEMA,
  TRIGGER_NAME,
  EVENT_MANIPULATION,
  EVENT_OBJECT_TABLE,
  ACTION_TIMING,
  ACTION_ORDER,
  CREATED,
  DEFINER
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
ORDER BY EVENT_OBJECT_TABLE, ACTION_TIMING, EVENT_MANIPULATION;
```

## Key Columns

| Column | Description |
|--------|-------------|
| `TRIGGER_NAME` | Trigger name |
| `EVENT_MANIPULATION` | INSERT, UPDATE, or DELETE |
| `EVENT_OBJECT_TABLE` | Table the trigger fires on |
| `ACTION_TIMING` | BEFORE or AFTER |
| `ACTION_ORDER` | Order among multiple triggers on same event |
| `ACTION_STATEMENT` | Trigger body SQL |
| `ACTION_ORIENTATION` | ROW (always in MySQL) |
| `DEFINER` | user@host of creator |
| `CREATED` | Creation timestamp |

## Listing All Triggers for a Table

```sql
SELECT
  TRIGGER_NAME,
  ACTION_TIMING,
  EVENT_MANIPULATION,
  ACTION_ORDER
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
  AND EVENT_OBJECT_TABLE = 'orders'
ORDER BY ACTION_TIMING, EVENT_MANIPULATION, ACTION_ORDER;
```

## Finding Tables with Multiple Triggers on the Same Event

MySQL allows multiple triggers per table per event (added in MySQL 5.7). Watch for ordering dependencies:

```sql
SELECT
  EVENT_OBJECT_TABLE,
  ACTION_TIMING,
  EVENT_MANIPULATION,
  COUNT(*) AS trigger_count
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
GROUP BY EVENT_OBJECT_TABLE, ACTION_TIMING, EVENT_MANIPULATION
HAVING trigger_count > 1
ORDER BY EVENT_OBJECT_TABLE;
```

## Reading a Trigger Definition

```sql
SELECT ACTION_STATEMENT
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
  AND TRIGGER_NAME = 'after_order_insert'\G
```

## Finding All BEFORE Triggers (Pre-validation Logic)

```sql
SELECT
  TRIGGER_NAME,
  EVENT_OBJECT_TABLE,
  EVENT_MANIPULATION,
  DEFINER
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
  AND ACTION_TIMING = 'BEFORE'
ORDER BY EVENT_OBJECT_TABLE, EVENT_MANIPULATION;
```

## Generating DROP TRIGGER Statements

Useful for schema cleanup or migration preparation:

```sql
SELECT
  CONCAT('DROP TRIGGER IF EXISTS `', TRIGGER_SCHEMA, '`.`', TRIGGER_NAME, '`;') AS drop_stmt
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
ORDER BY TRIGGER_NAME;
```

## Finding Recently Created Triggers

```sql
SELECT
  TRIGGER_NAME,
  EVENT_OBJECT_TABLE,
  EVENT_MANIPULATION,
  CREATED
FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'myapp'
  AND CREATED > DATE_SUB(NOW(), INTERVAL 30 DAY)
ORDER BY CREATED DESC;
```

## Summary

`INFORMATION_SCHEMA.TRIGGERS` exposes all database triggers including their timing, events, and SQL logic. Querying this table helps you audit hidden business logic, find tables with complex trigger chains, identify definer security concerns, and prepare for migrations by mapping trigger dependencies to table structure changes.
