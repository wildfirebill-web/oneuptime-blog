# How to Minimize Lock Contention in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Lock Contention, Performance, Concurrency

Description: Practical strategies to minimize lock contention in MySQL, including transaction design, indexing, and isolation level choices that improve throughput.

---

## Why Lock Contention Hurts Performance

Lock contention occurs when transactions compete for the same rows or resources, forcing some transactions to wait. High contention results in increased query latency, thread pile-up, and in extreme cases, cascading timeouts or deadlocks.

Understanding the root causes of contention - long transactions, missing indexes, or poor transaction design - is the first step to solving it.

## Strategy 1: Keep Transactions Short

The longer a transaction runs, the longer it holds locks. Commit as soon as possible after finishing the necessary work.

```sql
-- BAD: long transaction holding locks
START TRANSACTION;
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
-- application-side processing takes 5 seconds here
UPDATE orders SET status = 'processing' WHERE order_id = 123;
COMMIT;

-- BETTER: minimize the locked window
-- Do all preparation outside the transaction
-- Only enter the transaction for the actual write
START TRANSACTION;
UPDATE orders SET status = 'processing' WHERE order_id = 123;
COMMIT;
```

## Strategy 2: Use Proper Indexes

Without the right index, InnoDB scans more rows and places locks on rows it does not actually need to modify. Always ensure the WHERE clause in UPDATE and DELETE statements uses an indexed column.

```sql
-- Check if the query uses an index
EXPLAIN UPDATE orders SET status = 'shipped' WHERE customer_id = 42 AND status = 'pending';

-- Create a composite index if needed
ALTER TABLE orders ADD INDEX idx_customer_status (customer_id, status);
```

## Strategy 3: Process in Small Batches

When updating or deleting large sets of rows, process them in small batches rather than a single large transaction. This limits the lock duration and the number of rows locked simultaneously.

```sql
-- BAD: one giant delete locks all matched rows for the entire operation
DELETE FROM audit_logs WHERE created_at < '2024-01-01';

-- BETTER: batch deletes to reduce lock duration
DELETE FROM audit_logs WHERE created_at < '2024-01-01' LIMIT 1000;
-- Repeat until affected rows = 0
```

A simple loop in a stored procedure or application code:

```sql
DELIMITER $$
CREATE PROCEDURE cleanup_audit_logs()
BEGIN
  DECLARE done INT DEFAULT 0;
  REPEAT
    DELETE FROM audit_logs WHERE created_at < '2024-01-01' LIMIT 1000;
    SET done = (ROW_COUNT() = 0);
    DO SLEEP(0.1);  -- small pause to reduce pressure
  UNTIL done END REPEAT;
END$$
DELIMITER ;
```

## Strategy 4: Choose the Right Isolation Level

The default `REPEATABLE READ` isolation level uses gap locks and next-key locks to prevent phantom reads. If your application can tolerate phantom reads, switching to `READ COMMITTED` removes gap locks and reduces contention.

```sql
-- Check current isolation level
SELECT @@transaction_isolation;

-- Set for a single session
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Or set globally in my.cnf
-- transaction_isolation = READ-COMMITTED
```

## Strategy 5: Use Optimistic Locking

Rather than acquiring an exclusive lock at read time with `FOR UPDATE`, optimistic locking reads without a lock and uses a version column to detect conflicts at write time.

```sql
-- Schema with version column
ALTER TABLE accounts ADD COLUMN version INT NOT NULL DEFAULT 0;

-- Read without locking
SELECT balance, version FROM accounts WHERE account_id = 1;
-- application stores balance and version

-- Write: only update if version hasn't changed
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE account_id = 1 AND version = 5;

-- If ROW_COUNT() = 0, someone else updated - retry the operation
```

## Strategy 6: Avoid Hotspots

Hotspot rows are rows that many concurrent transactions compete to modify. A common example is a global counter or an inventory total.

```sql
-- BAD: all transactions update the same row
UPDATE global_stats SET total_orders = total_orders + 1;

-- BETTER: use a sharded counter table
-- Insert into one of N slots, aggregate with SUM
INSERT INTO order_counter_shards (shard_id, count)
VALUES (FLOOR(RAND() * 16), 1)
ON DUPLICATE KEY UPDATE count = count + 1;

-- Read the total
SELECT SUM(count) FROM order_counter_shards;
```

## Monitoring Contention

```sql
-- Lock waits per table
SELECT object_schema, object_name,
       count_read, count_write,
       sum_timer_wait / 1e12 AS total_wait_secs
FROM performance_schema.table_lock_waits_summary_by_table
ORDER BY sum_timer_wait DESC
LIMIT 20;

-- Current lock waits with blocking query
SELECT waiting_query, blocking_query, wait_age
FROM sys.innodb_lock_waits;
```

## Summary

Minimizing lock contention in MySQL requires a combination of transaction design, proper indexing, and choosing the right isolation level. Keep transactions short and focused, use indexes to limit the scope of locks, process large operations in batches, and consider optimistic locking or sharded counters for high-contention scenarios. Continuous monitoring with performance_schema helps identify and address new contention hotspots as your application evolves.
