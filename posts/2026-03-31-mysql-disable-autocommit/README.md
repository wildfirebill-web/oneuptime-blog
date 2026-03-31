# How to Disable Autocommit in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, InnoDB, Autocommit, Performance

Description: Learn how to disable autocommit in MySQL for session and global scope, the performance benefits, and important gotchas to avoid.

---

## Why Disable Autocommit?

Disabling autocommit allows you to execute multiple SQL statements as a single atomic unit before committing. This is essential for:

- Multi-step transactions that must succeed or fail together
- Bulk insert/update performance (fewer disk fsyncs)
- Explicit control over when changes become visible to other sessions

## Disabling Autocommit for the Current Session

```sql
-- Disable for current session only
SET SESSION autocommit = 0;
-- or equivalently
SET autocommit = 0;

-- Verify
SELECT @@autocommit;
-- Returns: 0
```

After this, every statement is automatically part of an implicit transaction until you explicitly COMMIT or ROLLBACK.

## Disabling Autocommit Globally

```sql
-- Affects all new connections (not existing ones)
SET GLOBAL autocommit = 0;

-- Persist across server restarts - add to my.cnf
```

```ini
[mysqld]
autocommit = 0
```

## Behavior with Autocommit Disabled

```sql
SET autocommit = 0;

-- These are now in an implicit transaction
INSERT INTO orders (customer_id, amount) VALUES (1, 99.99);
INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 5);

-- Must explicitly commit or rollback
COMMIT;
-- Both INSERTs are now permanent

-- If you just disconnect without committing, MySQL rolls back automatically
```

## Performance Benefit: Bulk Inserts

Each COMMIT flushes the InnoDB redo log to disk. Disabling autocommit for bulk operations dramatically reduces I/O:

```sql
SET autocommit = 0;

-- Insert 10,000 rows with a single commit
INSERT INTO logs (message, created_at) VALUES ('event 1', NOW());
INSERT INTO logs (message, created_at) VALUES ('event 2', NOW());
-- ... repeat 10,000 times

COMMIT;
-- Only ONE redo log flush instead of 10,000
```

Benchmark:

```bash
# With autocommit ON: ~1000 inserts/second (disk-bound)
# With autocommit OFF + single commit: ~50,000 inserts/second
```

## Common Gotcha: DDL Statements Cause Implicit Commits

Even with autocommit disabled, DDL statements automatically commit any pending transaction:

```sql
SET autocommit = 0;

INSERT INTO orders (customer_id, amount) VALUES (1, 99.99);

-- This implicitly commits the INSERT above!
ALTER TABLE products ADD COLUMN description TEXT;

-- The INSERT cannot be rolled back
ROLLBACK;
-- Only rolls back changes after the ALTER TABLE
```

Always commit or rollback pending transactions before running DDL.

## Using Application Connection Pools

Most connection pool libraries allow setting autocommit per connection:

```java
// Java - HikariCP
HikariConfig config = new HikariConfig();
config.setAutoCommit(false);  // Disable for all pooled connections
HikariDataSource ds = new HikariDataSource(config);

// Use explicit commit/rollback
Connection conn = ds.getConnection();
try {
    conn.createStatement().execute("UPDATE accounts SET balance = 500 WHERE id = 1");
    conn.commit();
} catch (Exception e) {
    conn.rollback();
}
```

## Re-enabling Autocommit

```sql
-- Re-enable for current session
SET autocommit = 1;

-- Any pending transaction is committed first
```

## Idle Connections Hold Open Transactions

A risk with disabled autocommit: idle connections in a pool hold open transactions that block undo log purge:

```sql
-- Detect idle open transactions
SELECT trx_id, trx_state, trx_mysql_thread_id
FROM information_schema.INNODB_TRX
WHERE trx_state = 'RUNNING'
  AND trx_query IS NULL;
```

Ensure your application always commits or rolls back before returning a connection to the pool.

## Summary

Disabling autocommit in MySQL gives full control over transaction boundaries and provides significant performance benefits for bulk operations by batching multiple statements into a single redo log flush. The key risks are forgetting to commit (leaving open transactions), unexpected implicit commits from DDL statements, and idle pooled connections holding open transactions.
