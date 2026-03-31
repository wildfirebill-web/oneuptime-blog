# How to Use SQL_BUFFER_RESULT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL_BUFFER_RESULT, Query, Lock

Description: Learn how SQL_BUFFER_RESULT works in MySQL to release table locks early by buffering result sets into a temporary table before sending rows to the client.

---

## What Is SQL_BUFFER_RESULT

`SQL_BUFFER_RESULT` is a MySQL-specific `SELECT` modifier that forces MySQL to materialize the entire result set into a temporary table before sending any rows to the client. Once the result is buffered, MySQL releases the read locks on the underlying tables, allowing other queries to modify those tables while the client reads the buffered result.

## Syntax

```sql
SELECT SQL_BUFFER_RESULT col1, col2, col3
FROM your_table
WHERE condition;
```

## When to Use SQL_BUFFER_RESULT

`SQL_BUFFER_RESULT` is useful in specific situations:

1. **Long result sets with slow clients** - If a client reads rows slowly, MySQL holds shared locks on source tables for the entire transfer duration. Buffering releases those locks early.
2. **Reducing lock contention** - On tables with heavy write traffic, long-running reads can block writes. Buffering completes the read quickly, freeing locks for writers.
3. **Client-side streaming with large results** - When streaming large results to a client application, buffering ensures the database I/O completes quickly.

## Example: Releasing Locks Early

Without `SQL_BUFFER_RESULT`, a slow client reading a large result holds shared locks throughout:

```sql
-- Without buffering: locks held during entire client transfer
SELECT id, name, email, notes
FROM customers
WHERE region = 'EMEA'
ORDER BY created_at;

-- With buffering: MySQL fills temp table quickly, releases locks,
-- then streams from temp table to client
SELECT SQL_BUFFER_RESULT id, name, email, notes
FROM customers
WHERE region = 'EMEA'
ORDER BY created_at;
```

## Verifying Temporary Table Creation

Use `EXPLAIN` to see the difference in the execution plan:

```sql
-- Check if a temporary table is used
EXPLAIN SELECT SQL_BUFFER_RESULT id, name
FROM orders
WHERE status = 'PENDING'\G
```

When `SQL_BUFFER_RESULT` is active, the `Extra` column in EXPLAIN will show `Using temporary`.

## Practical Example with Lock Monitoring

Observe lock behavior before and after:

```sql
-- Session 1: start a slow read without buffering
SELECT id, total FROM orders WHERE status = 'COMPLETED' ORDER BY total DESC;

-- Session 2 (concurrent): check locks held
SELECT object_name, lock_type, lock_status
FROM performance_schema.data_locks
WHERE object_name = 'orders';
```

With `SQL_BUFFER_RESULT` in Session 1, the locks are released as soon as the temporary table is populated, which is typically much faster than the client consuming all rows.

## Limitations

`SQL_BUFFER_RESULT` has trade-offs:

- **Memory and disk use** - The full result set is written to a temporary table. Large results can consume significant memory or spill to disk (`tmp_table_size` and `max_heap_table_size` control this).
- **No benefit for small results** - For small result sets, the overhead of creating a temporary table outweighs any lock reduction benefit.
- **Not a substitute for proper indexing** - It does not speed up the underlying query; it only changes when locks are released.

## Configuration for Large Buffered Results

```ini
[mysqld]
# Allow larger in-memory temporary tables before spilling to disk
tmp_table_size       = 256M
max_heap_table_size  = 256M
# If results exceed these limits, MySQL writes to disk
# Ensure tmpdir has enough space
tmpdir = /var/lib/mysql-tmp
```

## Alternative: Cover Reads with Transactions

For InnoDB tables, using a consistent snapshot transaction can reduce locking concerns without buffering:

```sql
START TRANSACTION WITH CONSISTENT SNAPSHOT;
SELECT id, name FROM customers WHERE region = 'EMEA' ORDER BY created_at;
COMMIT;
```

InnoDB's MVCC means this read does not block writers at the `REPEATABLE READ` isolation level.

## Summary

`SQL_BUFFER_RESULT` tells MySQL to fully materialize a query result into a temporary table before streaming rows to the client. This releases table locks earlier, reducing contention on busy write tables when a client reads results slowly. It is most valuable for large result sets with slow clients; for smaller queries or InnoDB tables the MVCC consistent snapshot is usually a better approach.
