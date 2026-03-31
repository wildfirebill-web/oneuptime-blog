# How to Monitor Active Transactions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Transaction, Monitoring, Performance

Description: Learn how to monitor active transactions in MySQL using INFORMATION_SCHEMA, performance_schema, and SHOW ENGINE INNODB STATUS.

---

## Why Monitor Active Transactions?

Active transactions that run for too long can hold locks, bloat the InnoDB undo log, block other queries, and degrade overall database performance. Regularly monitoring active transactions helps identify long-running sessions, find lock contention, and prevent resource exhaustion.

## Using INNODB_TRX View

The `information_schema.INNODB_TRX` view shows all currently active transactions:

```sql
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_mysql_thread_id AS thread_id,
    trx_query,
    trx_rows_locked,
    trx_rows_modified,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_seconds
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;
```

Key columns:
- `trx_state` - RUNNING, LOCK WAIT, ROLLING BACK, or COMMITTING
- `trx_started` - when the transaction began
- `trx_rows_locked` - number of rows currently locked
- `trx_query` - the most recent SQL statement

## Finding Blocking Transactions

```sql
-- Find which transaction is blocking which
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_seconds
FROM information_schema.INNODB_TRX AS r
JOIN performance_schema.data_lock_waits dlw
    ON r.trx_id = dlw.REQUESTING_ENGINE_TRANSACTION_ID
JOIN information_schema.INNODB_TRX AS b
    ON b.trx_id = dlw.BLOCKING_ENGINE_TRANSACTION_ID;
```

## Using performance_schema for Lock Details

```sql
-- View all currently held locks
SELECT
    DL.OBJECT_NAME AS table_name,
    DL.LOCK_TYPE,
    DL.LOCK_MODE,
    DL.LOCK_STATUS,
    T.trx_state,
    T.trx_mysql_thread_id AS thread_id
FROM performance_schema.data_locks DL
JOIN information_schema.INNODB_TRX T
    ON DL.ENGINE_TRANSACTION_ID = T.trx_id
ORDER BY T.trx_started;
```

## Using SHOW ENGINE INNODB STATUS

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `TRANSACTIONS` section which shows:

```text
---TRANSACTION 1234567, ACTIVE 45 sec
MySQL thread id 89, OS thread handle 12345, query id 99 localhost app
UPDATE orders SET status = 'shipped' WHERE id = 101
1 lock struct(s), heap size 1136, 1 row lock(s)
```

## Setting Up an Alert for Long-Running Transactions

Create a query to detect transactions running over a threshold:

```sql
SELECT
    trx_id,
    trx_mysql_thread_id AS thread_id,
    trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_seconds,
    trx_rows_locked,
    trx_query
FROM information_schema.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60
ORDER BY age_seconds DESC;
```

Schedule this as a periodic monitoring query in your alerting system.

## Monitoring with Performance Schema Summary Tables

```sql
-- Transaction summary by user
SELECT
    user,
    SUM_TIMER_WAIT / 1e12 AS total_wait_sec,
    COUNT_STAR AS transaction_count
FROM performance_schema.events_transactions_summary_by_user_by_event_name
ORDER BY total_wait_sec DESC
LIMIT 10;
```

## Killing a Stuck Transaction

Once you identify a problematic transaction by its thread ID:

```sql
-- Kill the connection (rolls back the transaction)
KILL CONNECTION <thread_id>;

-- Kill just the current query (leaves the connection open)
KILL QUERY <thread_id>;
```

## Summary

Monitoring active transactions in MySQL involves querying `information_schema.INNODB_TRX`, `performance_schema.data_locks`, and running `SHOW ENGINE INNODB STATUS`. Regularly checking for transactions running beyond a time threshold helps catch lock contention early, prevent undo log bloat, and maintain overall database health.
