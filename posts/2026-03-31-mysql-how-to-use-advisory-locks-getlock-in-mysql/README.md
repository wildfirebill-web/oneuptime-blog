# How to Use Advisory Locks (GET_LOCK) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Advisory Locks, GET_LOCK, Concurrency, Distributed Locking

Description: Learn how MySQL advisory locks with GET_LOCK work, how to use them for distributed locking, and their limitations compared to other locking mechanisms.

---

## What Are Advisory Locks

Advisory locks (also called named locks or user-level locks) are application-level mutexes implemented in MySQL. Unlike row locks or table locks that automatically protect data, advisory locks must be explicitly acquired and released by your application. MySQL does not enforce them on data - they are just named tokens for coordination.

Key use cases:
- Preventing duplicate processing in distributed systems
- Coordinating access to external resources
- Implementing distributed mutexes without a dedicated lock service
- Ensuring only one process runs a scheduled job at a time

## GET_LOCK: Acquiring a Lock

```sql
SELECT GET_LOCK('lock_name', timeout);
```

- `lock_name`: Arbitrary string name for the lock (max 64 characters)
- `timeout`: Seconds to wait; -1 waits indefinitely; 0 returns immediately

Return values:
- `1`: Lock acquired successfully
- `0`: Timeout expired, lock not acquired
- `NULL`: Error occurred

```sql
-- Acquire a lock, wait up to 10 seconds
SELECT GET_LOCK('job_nightly_report', 10);
-- Returns 1 if acquired, 0 if timed out

-- Try immediately without waiting
SELECT GET_LOCK('resource_export', 0);
```

## RELEASE_LOCK: Releasing a Lock

```sql
SELECT RELEASE_LOCK('lock_name');
```

Return values:
- `1`: Lock was released successfully
- `0`: Lock was not held by this connection
- `NULL`: Lock does not exist

```sql
SELECT RELEASE_LOCK('job_nightly_report');
```

## IS_FREE_LOCK and IS_USED_LOCK: Checking Lock State

```sql
-- Returns 1 if the lock is free (not held), 0 if held
SELECT IS_FREE_LOCK('job_nightly_report');

-- Returns the connection ID holding the lock, or NULL if free
SELECT IS_USED_LOCK('job_nightly_report');
```

## Practical Example: Single-Instance Job Scheduler

A common pattern is ensuring only one instance of a job runs at a time across multiple application servers:

```python
import mysql.connector
import time

def run_nightly_report_with_lock():
    conn = mysql.connector.connect(
        host='localhost', database='app', user='appuser', password='secret'
    )
    cursor = conn.cursor()

    try:
        # Try to acquire the lock (wait up to 0 seconds - non-blocking)
        cursor.execute("SELECT GET_LOCK('nightly_report_job', 0)")
        result = cursor.fetchone()[0]

        if result != 1:
            print("Another instance is running, skipping.")
            return

        print("Lock acquired, running nightly report...")

        # Simulate job work
        cursor.execute("CALL generate_nightly_report()")
        conn.commit()
        print("Report completed successfully.")

    finally:
        # Always release the lock
        cursor.execute("SELECT RELEASE_LOCK('nightly_report_job')")
        cursor.close()
        conn.close()

run_nightly_report_with_lock()
```

## Multiple Locks Per Connection

A single connection can hold multiple advisory locks:

```sql
SELECT GET_LOCK('module_a', 5);  -- Acquire lock A
SELECT GET_LOCK('module_b', 5);  -- Acquire lock B while holding A

-- Release individually
SELECT RELEASE_LOCK('module_a');
SELECT RELEASE_LOCK('module_b');

-- Or release all at once
SELECT RELEASE_ALL_LOCKS();
```

## Automatic Lock Release

Advisory locks are automatically released when:
- The connection closes (gracefully or via crash)
- `RELEASE_LOCK()` is called explicitly
- `RELEASE_ALL_LOCKS()` is called

This automatic release on disconnect is important for reliability - if your application crashes while holding a lock, it is released immediately when MySQL detects the disconnection.

## Deadlock Risk with Multiple Locks

Advisory locks can cause deadlocks if multiple connections try to acquire locks in different orders:

```sql
-- Connection 1:
SELECT GET_LOCK('lock_A', 10);
SELECT GET_LOCK('lock_B', 10);  -- Waits for Connection 2

-- Connection 2:
SELECT GET_LOCK('lock_B', 10);
SELECT GET_LOCK('lock_A', 10);  -- Waits for Connection 1 = DEADLOCK
```

To avoid deadlocks, always acquire multiple locks in a consistent, alphabetically ordered sequence.

```sql
-- Safe: always acquire in alphabetical order
SELECT GET_LOCK('lock_A', 10);
SELECT GET_LOCK('lock_B', 10);
```

## Stored Procedure with Advisory Lock

```sql
DELIMITER //
CREATE PROCEDURE run_exclusive_task(IN p_task_name VARCHAR(64), IN p_timeout INT)
BEGIN
  DECLARE v_lock_result INT;

  SELECT GET_LOCK(p_task_name, p_timeout) INTO v_lock_result;

  IF v_lock_result = 1 THEN
    -- Execute the protected task
    CALL do_actual_work();

    -- Release lock
    SELECT RELEASE_LOCK(p_task_name) INTO @released;
  ELSE
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'Could not acquire advisory lock - task already running';
  END IF;
END //
DELIMITER ;
```

## Limitations of Advisory Locks

```text
- Not replicated: advisory locks exist only on the server where they were acquired.
  Do not use for cross-replica coordination.
- Per-connection: a connection cannot acquire the same lock name twice
  (second GET_LOCK replaces the first).
- Not visible in INFORMATION_SCHEMA: unlike table locks, advisory locks
  do not appear in lock monitoring views.
- 64-character name limit: plan your naming convention accordingly.
```

## Advisory Locks vs Redis Locks

For high-throughput distributed locking, Redis `SET NX PX` or Redlock may be more appropriate:

```python
import redis

r = redis.Redis(host='localhost')

# Redis advisory lock with TTL
acquired = r.set('my_lock', '1', nx=True, ex=30)  # NX=only if not exists, ex=30s TTL
if acquired:
    try:
        # Do work
        pass
    finally:
        r.delete('my_lock')
```

MySQL advisory locks are sufficient for lower-throughput coordination where you already have a MySQL connection and don't want to introduce another dependency.

## Summary

MySQL advisory locks via `GET_LOCK()` provide a simple distributed mutex mechanism using named string tokens. They are automatically released on connection close, support configurable wait timeouts, and can be held in multiples per connection. They are ideal for preventing duplicate job execution across application servers but are limited to single-server scope (not replicated) and require careful ordering when acquiring multiple locks to avoid deadlocks.
