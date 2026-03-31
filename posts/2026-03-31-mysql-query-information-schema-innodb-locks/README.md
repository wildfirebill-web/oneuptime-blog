# How to Query INFORMATION_SCHEMA.INNODB_LOCKS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, InnoDB, Lock, Deadlock

Description: Learn how to query INFORMATION_SCHEMA.INNODB_LOCKS in MySQL 5.7 and the Performance Schema data_locks table in MySQL 8.0 to diagnose lock contention.

---

## Overview

Lock visibility is critical for diagnosing deadlocks and blocking in MySQL. In MySQL 5.7 and earlier, `INFORMATION_SCHEMA.INNODB_LOCKS` showed all locks being held or requested. In MySQL 8.0, this view was replaced by `performance_schema.data_locks`, which provides the same information with more detail.

## MySQL 5.7: INNODB_LOCKS

```sql
-- MySQL 5.7 only
SELECT
  LOCK_ID,
  LOCK_TRX_ID,
  LOCK_MODE,
  LOCK_TYPE,
  LOCK_TABLE,
  LOCK_INDEX,
  LOCK_SPACE,
  LOCK_PAGE,
  LOCK_REC,
  LOCK_DATA
FROM information_schema.INNODB_LOCKS
ORDER BY LOCK_TRX_ID;
```

## MySQL 8.0: performance_schema.data_locks

```sql
-- MySQL 8.0+
SELECT
  ENGINE_TRANSACTION_ID,
  THREAD_ID,
  OBJECT_SCHEMA,
  OBJECT_NAME,
  INDEX_NAME,
  LOCK_TYPE,
  LOCK_MODE,
  LOCK_STATUS,
  LOCK_DATA
FROM performance_schema.data_locks
ORDER BY ENGINE_TRANSACTION_ID;
```

## Lock Mode Reference

| Lock Mode | Description |
|-----------|-------------|
| `S` | Shared lock (read lock) |
| `X` | Exclusive lock (write lock) |
| `IS` | Intention shared (table-level intent) |
| `IX` | Intention exclusive (table-level intent) |
| `S,GAP` | Shared gap lock |
| `X,GAP` | Exclusive gap lock |
| `X,REC_NOT_GAP` | Record lock only (no gap) |
| `X,INSERT_INTENTION` | Insert intention lock |

## Finding Who is Holding Locks on a Specific Table

```sql
-- MySQL 8.0
SELECT
  t.PROCESSLIST_USER,
  t.PROCESSLIST_HOST,
  l.LOCK_TYPE,
  l.LOCK_MODE,
  l.LOCK_STATUS,
  l.LOCK_DATA
FROM performance_schema.data_locks l
JOIN performance_schema.threads t
  ON l.THREAD_ID = t.THREAD_ID
WHERE l.OBJECT_NAME = 'orders'
ORDER BY l.LOCK_STATUS DESC;
```

## Counting Locks Per Transaction

```sql
-- MySQL 8.0
SELECT
  ENGINE_TRANSACTION_ID,
  COUNT(*) AS lock_count,
  COUNT(DISTINCT OBJECT_NAME) AS tables_locked,
  SUM(LOCK_STATUS = 'WAITING') AS waiting_locks
FROM performance_schema.data_locks
GROUP BY ENGINE_TRANSACTION_ID
ORDER BY lock_count DESC;
```

## Full Blocking Chain Analysis

```sql
-- MySQL 8.0: find blocker and waiter
SELECT
  r.ENGINE_TRANSACTION_ID AS waiting_trx,
  r.OBJECT_NAME AS locked_table,
  r.LOCK_MODE AS waiting_mode,
  b.ENGINE_TRANSACTION_ID AS blocking_trx,
  b.LOCK_MODE AS blocking_mode
FROM performance_schema.data_locks r
JOIN performance_schema.data_locks b
  ON r.OBJECT_NAME = b.OBJECT_NAME
  AND r.LOCK_STATUS = 'WAITING'
  AND b.LOCK_STATUS = 'GRANTED';
```

## Summary

Lock monitoring in MySQL requires different approaches depending on version. In MySQL 5.7, use `INFORMATION_SCHEMA.INNODB_LOCKS` for basic lock inspection. In MySQL 8.0, use `performance_schema.data_locks` for comprehensive lock analysis including gap locks, record locks, and intention locks. Combine with `INNODB_TRX` and `data_lock_waits` to build a complete picture of lock contention chains.
