# How to Use IS_FREE_LOCK() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, IS_FREE_LOCK, Advisory Locks, Concurrency, SQL Functions

Description: Learn how to use the IS_FREE_LOCK() function in MySQL to check if a named advisory lock is available without attempting to acquire it.

---

## What is IS_FREE_LOCK()

MySQL's `IS_FREE_LOCK()` function checks whether a named advisory lock is currently available (not held by any session). It is a non-blocking inspection function used alongside `GET_LOCK()` and `RELEASE_LOCK()`.

Syntax:

```sql
IS_FREE_LOCK(lock_name)
```

Returns:
- `1` - the lock is free (not held by any session)
- `0` - the lock is currently held by a session
- `NULL` - the lock name is invalid or an error occurred

## Basic Usage

```sql
-- Check if a lock is free before trying to acquire it
SELECT IS_FREE_LOCK('batch_job_lock');
-- Output: 1 (free) or 0 (held)
```

## IS_FREE_LOCK() vs IS_USED_LOCK()

```sql
-- IS_FREE_LOCK: returns 1 if free, 0 if held
SELECT IS_FREE_LOCK('my_lock');

-- IS_USED_LOCK: returns the connection ID of the holder, or NULL if free
SELECT IS_USED_LOCK('my_lock');
```

Use `IS_FREE_LOCK()` when you only need a boolean answer. Use `IS_USED_LOCK()` when you need to know which connection holds the lock.

## Checking Lock Status Before Acquisition

```sql
-- First check, then conditionally acquire
SELECT IS_FREE_LOCK('report_lock') AS is_free;

-- If free (1), acquire it
SELECT GET_LOCK('report_lock', 0) AS acquired;
```

Note: The check and acquisition are not atomic. Another session could acquire the lock between the two calls. For atomic acquire, use `GET_LOCK()` directly with a timeout of 0:

```sql
-- Atomic non-blocking try-acquire (preferred)
SELECT GET_LOCK('report_lock', 0) AS acquired;
-- Returns 1 if acquired, 0 if already held
```

## Using IS_FREE_LOCK() for Monitoring

List all named locks and their status using the performance schema:

```sql
SELECT
  object_name AS lock_name,
  owner_thread_id,
  owner_event_id
FROM performance_schema.metadata_locks
WHERE object_type = 'USER LEVEL LOCK';
```

Or check specific known lock names:

```sql
SELECT
  'batch_import' AS lock_name,
  IS_FREE_LOCK('batch_import') AS is_free,
  IS_USED_LOCK('batch_import') AS held_by_connection;
```

## IS_FREE_LOCK() in a Decision Procedure

```sql
DELIMITER //
CREATE PROCEDURE check_and_report_lock(IN p_lock_name VARCHAR(64))
BEGIN
  DECLARE lock_status INT;
  DECLARE holder_conn INT;

  SET lock_status = IS_FREE_LOCK(p_lock_name);
  SET holder_conn = IS_USED_LOCK(p_lock_name);

  IF lock_status = 1 THEN
    SELECT CONCAT(p_lock_name, ' is available') AS status;
  ELSE
    SELECT CONCAT(p_lock_name, ' is held by connection ', holder_conn) AS status;
  END IF;
END //
DELIMITER ;

CALL check_and_report_lock('batch_job_lock');
```

## Real-World Use Case - Coordinated Job Scheduling

In a system where multiple application servers compete to run a scheduled job:

```python
def should_run_job(cursor, job_name):
    """Check if job lock is available before trying to acquire."""
    cursor.execute("SELECT IS_FREE_LOCK(%s)", (job_name,))
    is_free = cursor.fetchone()[0]

    if not is_free:
        print(f"Job '{job_name}' is already running on another server")
        return False

    # Try to acquire (non-blocking)
    cursor.execute("SELECT GET_LOCK(%s, 0)", (job_name,))
    acquired = cursor.fetchone()[0]

    if acquired:
        print(f"Acquired lock for '{job_name}'")
        return True
    else:
        print(f"Lock for '{job_name}' was acquired by another process just now")
        return False
```

## Lock Name Best Practices

```sql
-- Use descriptive, namespaced lock names
SELECT IS_FREE_LOCK('myapp.nightly_cleanup');
SELECT IS_FREE_LOCK('myapp.report_generator.daily');
SELECT IS_FREE_LOCK(CONCAT('user_export_', user_id));
```

Lock names can be up to 64 characters long (MySQL 5.7+).

## IS_FREE_LOCK() with Multiple Locks

Check multiple locks at once:

```sql
SELECT
  IS_FREE_LOCK('lock_a') AS lock_a_free,
  IS_FREE_LOCK('lock_b') AS lock_b_free,
  IS_FREE_LOCK('lock_c') AS lock_c_free;
```

## Summary

`IS_FREE_LOCK(name)` returns 1 if a named advisory lock is not currently held, 0 if it is held, and NULL on error. It is a non-blocking inspection function useful for monitoring lock state and making conditional decisions before attempting acquisition. For exclusive access, always use `GET_LOCK()` directly - `IS_FREE_LOCK()` alone does not prevent race conditions between the check and the acquire. Use `IS_USED_LOCK()` when you need the connection ID of the lock holder for debugging.
