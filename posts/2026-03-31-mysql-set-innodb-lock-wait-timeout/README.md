# How to Set innodb_lock_wait_timeout in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Lock, Transaction, Configuration

Description: Learn how to configure innodb_lock_wait_timeout in MySQL to control how long transactions wait for row locks before timing out.

---

## Overview

`innodb_lock_wait_timeout` controls how many seconds an InnoDB transaction waits for a row lock before giving up and returning an error. The default is 50 seconds. Setting the right value prevents transactions from waiting indefinitely while avoiding premature timeouts for legitimate long-running operations.

## Checking the Current Setting

```sql
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';
```

Default output:
```
+--------------------------+-------+
| Variable_name            | Value |
++--------------------------+-------+
| innodb_lock_wait_timeout | 50    |
+--------------------------+-------+
```

## Setting the Timeout for the Current Session

Adjust the timeout for a specific session without affecting other connections:

```sql
SET SESSION innodb_lock_wait_timeout = 10;
```

This is useful for batch jobs or imports where you want faster failure rather than long waits.

## Setting the Global Timeout

Change the default for all new connections:

```sql
SET GLOBAL innodb_lock_wait_timeout = 30;
```

This affects new connections but not existing ones. Existing sessions retain their previous value until reconnection.

## Making the Setting Permanent

To persist the setting across MySQL restarts, add it to `my.cnf` or `my.ini`:

```bash
[mysqld]
innodb_lock_wait_timeout = 30
```

Then restart MySQL:

```bash
sudo systemctl restart mysql
```

## What Happens When a Timeout Occurs

When a lock wait exceeds the timeout, the waiting transaction receives error 1205:

```
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

Only the timed-out statement is rolled back by default. The transaction remains open. You must explicitly roll it back:

```sql
ROLLBACK;
```

## Controlling Rollback Scope with innodb_rollback_on_timeout

By default, only the timed-out statement rolls back. To roll back the entire transaction on timeout:

```sql
-- Check current setting
SHOW VARIABLES LIKE 'innodb_rollback_on_timeout';

-- Enable in my.cnf (cannot be changed dynamically)
[mysqld]
innodb_rollback_on_timeout = ON
```

This is safer for applications that do not explicitly handle partial rollbacks.

## Recommended Values by Use Case

| Use Case | Recommended Timeout |
|----------|-------------------|
| OLTP web applications | 5-15 seconds |
| Batch processing | 30-60 seconds |
| Reporting queries | 120+ seconds |
| Development/testing | 5 seconds |

## Application-Level Retry on Timeout

Configure applications to retry on lock timeout errors:

```python
import mysql.connector
import time

def execute_with_retry(cursor, query, max_retries=3):
    for attempt in range(max_retries):
        try:
            cursor.execute(query)
            return
        except mysql.connector.Error as e:
            if e.errno == 1205 and attempt < max_retries - 1:
                time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
                continue
            raise e
```

## Diagnosing Lock Waits Before They Timeout

Monitor current lock waits to identify blocking transactions:

```sql
SELECT
  waiting_trx_id,
  waiting_query,
  blocking_trx_id,
  blocking_query
FROM sys.innodb_lock_waits;
```

If a transaction is blocking many others, consider killing it:

```sql
KILL <blocking_thread_id>;
```

## Monitoring Lock Timeouts with OneUptime

Configure OneUptime to track the MySQL error rate metric. A spike in 1205 errors indicates lock contention problems. Set up alerts to notify your team when timeouts exceed a threshold.

## Summary

`innodb_lock_wait_timeout` determines how long InnoDB waits for a row lock before returning error 1205. Set it per-session for specific workloads, globally for defaults, and persistently in `my.cnf` for production. Enable `innodb_rollback_on_timeout` to roll back entire transactions on timeout, and implement retry logic in applications to handle intermittent lock contention gracefully.
