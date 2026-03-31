# How to Detect Deadlocks in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Deadlock, InnoDB, Transaction, Monitoring

Description: Learn how to detect deadlocks in MySQL using SHOW ENGINE INNODB STATUS, performance_schema, and error logs to diagnose lock contention.

---

## Overview

A deadlock occurs when two or more transactions hold locks and each is waiting for the other to release its lock, creating a circular dependency. MySQL's InnoDB engine detects deadlocks automatically and rolls back the transaction with the smallest undo log. Understanding how to detect and diagnose deadlocks is essential for maintaining database reliability.

## How MySQL Detects Deadlocks

InnoDB uses a lock wait graph to detect cycles. When a deadlock is detected:
1. InnoDB selects a victim transaction (typically the one with the least undo data)
2. The victim is rolled back
3. The other transaction proceeds
4. Error 1213 is returned to the rolled-back transaction's client

## Detecting Deadlocks via SHOW ENGINE INNODB STATUS

The most detailed deadlock information is in the InnoDB status output:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `LATEST DETECTED DEADLOCK` section:

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-06-01 14:23:45 0x7f...
*** (1) TRANSACTION:
TRANSACTION 421938, ACTIVE 5 sec starting index read
...
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 47 page no 4 n bits 72 index PRIMARY of table `mydb`.`orders`
...
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
...
*** (2) TRANSACTION:
...
*** WE ROLL BACK TRANSACTION (2)
```

This shows both transactions, what locks each holds, and what each is waiting for.

## Detecting Deadlocks via Error Log

MySQL logs deadlocks in the error log when `innodb_print_all_deadlocks` is enabled:

```sql
-- Enable deadlock logging
SET GLOBAL innodb_print_all_deadlocks = ON;
```

Then monitor the MySQL error log:

```bash
tail -f /var/log/mysql/error.log | grep -A 30 "DEADLOCK"
```

## Detecting Deadlocks via performance_schema

Query the performance_schema for recent lock waits:

```sql
SELECT
  r.trx_id AS waiting_trx_id,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx_id,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id;
```

Note: In MySQL 8.0+, use `performance_schema.data_lock_waits`:

```sql
SELECT
  REQUESTING_ENGINE_TRANSACTION_ID,
  BLOCKING_ENGINE_TRANSACTION_ID,
  REQUESTING_THREAD_ID,
  BLOCKING_THREAD_ID
FROM performance_schema.data_lock_waits;
```

## Detecting Deadlocks via Application Errors

At the application level, deadlocks appear as exception code 1213:

```python
import mysql.connector

try:
    cursor.execute("UPDATE orders SET status = 'processing' WHERE id = 1")
except mysql.connector.Error as e:
    if e.errno == 1213:
        print("Deadlock detected - retrying transaction")
        # Implement retry logic
```

## Monitoring Deadlock Frequency

Track the cumulative deadlock count:

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';
```

Monitor this value over time to see deadlock frequency trends. A sudden increase indicates a problem.

## Monitoring Deadlocks with OneUptime

Configure OneUptime to alert when MySQL error rates spike. Deadlock-induced transaction failures increase application error rates - monitoring these gives early warning before deadlocks become a user-visible problem.

## Summary

Detect MySQL deadlocks using `SHOW ENGINE INNODB STATUS` for detailed diagnostics, `innodb_print_all_deadlocks` for continuous logging, and `performance_schema.data_lock_waits` for programmatic monitoring. Track the `Innodb_deadlocks` status variable to measure frequency over time and set up application-level retry logic for deadlock error 1213.
