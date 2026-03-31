# How to Fix ERROR 1205 Lock Wait Timeout Exceeded in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Lock Wait Timeout, Transaction, Troubleshooting

Description: Learn how to diagnose and fix MySQL ERROR 1205 Lock Wait Timeout Exceeded by identifying blocking transactions and tuning lock settings.

---

`ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction` occurs when a transaction waits longer than `innodb_lock_wait_timeout` seconds for a row lock held by another transaction.

## Understanding the Error

InnoDB uses row-level locking. When transaction A holds a lock on a row and transaction B tries to modify the same row, B waits. If B waits longer than `innodb_lock_wait_timeout` (default 50 seconds), it receives this error. Transaction A is NOT rolled back.

## Immediate Diagnosis

```sql
-- Find all waiting transactions
SELECT
    r.trx_id          AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query       AS waiting_query,
    b.trx_id          AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query       AS blocking_query
FROM       information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id
INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id;
```

```sql
-- MySQL 8.0+ uses performance_schema instead
SELECT
    waiting.THREAD_ID,
    waiting.SQL_TEXT   AS waiting_sql,
    blocking.THREAD_ID AS blocking_thread,
    blocking.SQL_TEXT  AS blocking_sql
FROM performance_schema.data_lock_waits dlw
JOIN performance_schema.events_statements_current waiting
    ON waiting.THREAD_ID = dlw.REQUESTING_THREAD_ID
JOIN performance_schema.events_statements_current blocking
    ON blocking.THREAD_ID = dlw.BLOCKING_THREAD_ID;
```

## Kill the Blocking Transaction

```sql
-- Kill the blocking thread (found from the query above)
KILL CONNECTION <blocking_thread_id>;
```

## Common Causes and Fixes

### Long-Running Uncommitted Transactions

An application that opens a transaction and never commits is the most frequent cause:

```sql
-- Check for old transactions
SELECT trx_id, trx_started, trx_query, trx_mysql_thread_id
FROM   information_schema.innodb_trx
WHERE  trx_started < NOW() - INTERVAL 30 SECOND;
```

Fix: ensure application code always calls `COMMIT` or `ROLLBACK` in a finally block.

### Missing Index on Updated Column

An UPDATE without an index does a full table scan and locks all rows it reads:

```sql
-- Bad: full table scan locks many rows
UPDATE orders SET status = 'shipped' WHERE customer_name = 'Alice';

-- Good: use an indexed column
UPDATE orders SET status = 'shipped' WHERE id = 42;
```

Fix: add an index on columns used in UPDATE/DELETE WHERE clauses.

## Tuning innodb_lock_wait_timeout

```sql
-- Reduce timeout so application gets errors faster and retries
SET GLOBAL innodb_lock_wait_timeout = 10;
```

Shorter timeouts surface problems faster and allow retry logic to kick in sooner.

## Application-Level Retry

```python
import mysql.connector, time

def update_with_retry(conn, order_id, retries=3):
    for attempt in range(retries):
        try:
            cursor = conn.cursor()
            cursor.execute("UPDATE orders SET status='shipped' WHERE id=%s", (order_id,))
            conn.commit()
            return
        except mysql.connector.Error as e:
            if e.errno == 1205 and attempt < retries - 1:
                time.sleep(0.5 * (attempt + 1))
                conn.rollback()
                continue
            raise
```

## Summary

ERROR 1205 is caused by one transaction blocking another. Diagnose using `information_schema.innodb_trx` and `data_lock_waits`. Kill the blocking transaction to unblock immediately. Prevent recurrence by always committing transactions promptly, indexing columns in UPDATE/DELETE WHERE clauses, and implementing application-level retry logic for transient lock errors.
