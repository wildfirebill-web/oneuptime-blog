# How to Query INFORMATION_SCHEMA.INNODB_LOCK_WAITS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, InnoDB, Lock Wait, Deadlock

Description: Learn how to query INFORMATION_SCHEMA.INNODB_LOCK_WAITS and Performance Schema data_lock_waits to diagnose blocking transaction chains in MySQL.

---

## Overview

`INFORMATION_SCHEMA.INNODB_LOCK_WAITS` (MySQL 5.7) and `performance_schema.data_lock_waits` (MySQL 8.0) show which transactions are waiting for locks and which transactions are blocking them. This is the key table for diagnosing deadlock scenarios and lock contention chains.

## MySQL 5.7: INNODB_LOCK_WAITS

```sql
-- MySQL 5.7 only
SELECT
  REQUESTING_TRX_ID,
  REQUESTED_LOCK_ID,
  BLOCKING_TRX_ID,
  BLOCKING_LOCK_ID
FROM information_schema.INNODB_LOCK_WAITS;
```

## MySQL 8.0: data_lock_waits

```sql
-- MySQL 8.0+
SELECT
  REQUESTING_ENGINE_TRANSACTION_ID,
  REQUESTING_THREAD_ID,
  BLOCKING_ENGINE_TRANSACTION_ID,
  BLOCKING_THREAD_ID
FROM performance_schema.data_lock_waits;
```

## Complete Blocking Chain Query (MySQL 8.0)

The most useful query combines three tables to identify the full chain:

```sql
SELECT
  r_t.PROCESSLIST_USER AS waiting_user,
  r_t.PROCESSLIST_HOST AS waiting_host,
  r_trx.TRX_QUERY AS waiting_query,
  TIMESTAMPDIFF(SECOND, r_trx.TRX_WAIT_STARTED, NOW()) AS wait_sec,
  b_t.PROCESSLIST_USER AS blocking_user,
  b_t.PROCESSLIST_HOST AS blocking_host,
  b_trx.TRX_QUERY AS blocking_query,
  TIMESTAMPDIFF(SECOND, b_trx.TRX_STARTED, NOW()) AS blocker_age_sec
FROM performance_schema.data_lock_waits lw
JOIN information_schema.INNODB_TRX r_trx
  ON lw.REQUESTING_ENGINE_TRANSACTION_ID = r_trx.TRX_ID
JOIN information_schema.INNODB_TRX b_trx
  ON lw.BLOCKING_ENGINE_TRANSACTION_ID = b_trx.TRX_ID
JOIN performance_schema.threads r_t
  ON lw.REQUESTING_THREAD_ID = r_t.THREAD_ID
JOIN performance_schema.threads b_t
  ON lw.BLOCKING_THREAD_ID = b_t.THREAD_ID;
```

## Finding the Root Blocker in a Chain

In long blocking chains, multiple transactions may be waiting on a single root blocker:

```sql
SELECT
  BLOCKING_ENGINE_TRANSACTION_ID AS blocker_trx,
  COUNT(*) AS waiters,
  GROUP_CONCAT(REQUESTING_ENGINE_TRANSACTION_ID) AS waiting_trxs
FROM performance_schema.data_lock_waits
GROUP BY BLOCKING_ENGINE_TRANSACTION_ID
ORDER BY waiters DESC;
```

## Killing the Blocker

After identifying the blocking connection:

```sql
-- Find the thread ID for the blocking transaction
SELECT TRX_MYSQL_THREAD_ID
FROM information_schema.INNODB_TRX
WHERE TRX_ID = 'blocking_trx_id_here';

-- Kill it
KILL 1234;
```

## Checking for Deadlock History

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `LATEST DETECTED DEADLOCK` section which shows the last deadlock cycle.

## Preventing Future Lock Waits

```sql
-- Check lock wait timeout setting
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';

-- Reduce to fail faster (default is 50 seconds)
SET GLOBAL innodb_lock_wait_timeout = 10;
```

## Summary

`INFORMATION_SCHEMA.INNODB_LOCK_WAITS` and its MySQL 8.0 replacement `performance_schema.data_lock_waits` expose the transaction blocking chain. By joining them with `INNODB_TRX` and `performance_schema.threads`, you can identify exactly which transaction is blocking others, how long the wait has been going on, and take immediate action to resolve the contention.
