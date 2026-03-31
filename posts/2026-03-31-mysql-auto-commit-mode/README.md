# How to Understand Auto-Commit Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transaction, InnoDB, Autocommit, Concurrency

Description: Understand how MySQL auto-commit mode works, how it affects transactions, and what happens when you enable or disable it.

---

## What Is Auto-Commit Mode?

By default, MySQL operates in auto-commit mode. When auto-commit is ON, every SQL statement that is not already part of an explicit transaction is treated as its own transaction and committed immediately after execution. There is no need to call `COMMIT` - changes are permanent as soon as the statement succeeds.

## Checking the Current Auto-Commit Setting

```sql
SELECT @@autocommit;
-- Returns 1 (ON) by default
```

Or:

```sql
SHOW VARIABLES LIKE 'autocommit';
-- Variable_name: autocommit, Value: ON
```

## How Auto-Commit Affects Statements

With auto-commit ON:

```sql
-- Each of these is automatically wrapped in a transaction and committed
INSERT INTO orders (customer_id, amount) VALUES (1, 99.99);
-- Committed immediately

UPDATE accounts SET balance = 500 WHERE id = 1;
-- Committed immediately

DELETE FROM temp_data WHERE id = 10;
-- Committed immediately
```

There is no chance to roll back once the statement completes.

## Explicit Transactions Override Auto-Commit

Using `START TRANSACTION` or `BEGIN` temporarily suspends auto-commit for the duration of that transaction:

```sql
-- Auto-commit is ON but explicit transaction takes precedence
START TRANSACTION;

INSERT INTO orders (customer_id, amount) VALUES (1, 99.99);
UPDATE accounts SET balance = balance - 99.99 WHERE id = 1;

-- Neither statement is committed yet
COMMIT;
-- Both committed together atomically

-- Auto-commit resumes after COMMIT or ROLLBACK
```

## Auto-Commit and Non-Transactional Statements

Some statements cannot be rolled back regardless of auto-commit setting:

```sql
-- These are auto-committed regardless of auto-commit mode
CREATE TABLE, DROP TABLE, ALTER TABLE, TRUNCATE TABLE
-- They also implicitly commit any open transaction
```

## Auto-Commit and Performance

Auto-commit ON with many small updates creates one transaction per statement. For bulk operations, wrapping in an explicit transaction is significantly faster:

```sql
-- Slow: 10,000 separate transactions
UPDATE stats SET views = views + 1 WHERE id = 1;
UPDATE stats SET views = views + 1 WHERE id = 2;
-- ... (10,000 individual commits)

-- Fast: One transaction for all updates
START TRANSACTION;
UPDATE stats SET views = views + 1 WHERE id IN (1,2,3,...);
COMMIT;
-- Or batch by looping and committing every 1000 rows
```

Each COMMIT involves a fsync of the InnoDB redo log, which is expensive. Batching reduces the number of fsyncs.

## Checking Auto-Commit in Application Drivers

Different drivers expose auto-commit settings differently:

```python
# Python - mysql-connector-python
conn = mysql.connector.connect(host='localhost', database='mydb',
                                user='root', password='pass')
conn.autocommit = False  # Disable auto-commit
cursor = conn.cursor()
cursor.execute("UPDATE accounts SET balance = 500 WHERE id = 1")
conn.commit()  # Must explicitly commit
```

```javascript
// Node.js - mysql2
const conn = await pool.getConnection();
await conn.beginTransaction();  // Disables auto-commit for this block
await conn.execute('UPDATE accounts SET balance = 500 WHERE id = 1');
await conn.commit();
```

## Auto-Commit and Replication

Auto-commit ON means each statement is replicated as a separate transaction to replicas. For heavy write workloads, this increases replication overhead compared to batched explicit transactions.

## Summary

MySQL auto-commit mode automatically commits each SQL statement as its own transaction. While convenient for simple one-off statements, it prevents multi-statement atomicity and is less efficient for bulk operations. Use explicit transactions with `START TRANSACTION` when you need to group multiple statements atomically or improve write performance through batching.
