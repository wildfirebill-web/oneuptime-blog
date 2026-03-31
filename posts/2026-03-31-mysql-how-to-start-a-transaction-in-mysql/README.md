# How to Start a Transaction in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Transactions, Innodb, Data Integrity

Description: Learn how to start a transaction in MySQL using START TRANSACTION or BEGIN, and understand autocommit behavior and transaction boundaries.

---

## What Is a Transaction?

A transaction is a logical unit of work that groups one or more SQL statements into an atomic operation. Either all statements succeed (and changes are committed) or all fail (and changes are rolled back). MySQL transactions follow the ACID properties: Atomicity, Consistency, Isolation, and Durability.

MySQL InnoDB is the storage engine that supports transactions. MyISAM does not support transactions.

## Starting a Transaction

There are two equivalent ways to start a transaction in MySQL:

```sql
START TRANSACTION;

-- or the shorter form
BEGIN;
```

Both begin a new transaction. All subsequent DML statements (`INSERT`, `UPDATE`, `DELETE`) are part of the transaction until you issue `COMMIT` or `ROLLBACK`.

## Basic Transaction Example

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE account_id = 2;

COMMIT;
```

Both updates are committed together. If either fails, neither takes effect if you roll back.

## Transaction with Error Handling

In application code, use try/catch/finally to handle transaction boundaries:

```javascript
const conn = await pool.getConnection();
await conn.beginTransaction();

try {
    await conn.execute(
        'UPDATE accounts SET balance = balance - ? WHERE account_id = ?',
        [500, 1]
    );
    await conn.execute(
        'UPDATE accounts SET balance = balance + ? WHERE account_id = ?',
        [500, 2]
    );
    await conn.commit();
} catch (err) {
    await conn.rollback();
    throw err;
} finally {
    conn.release();
}
```

## Understanding Autocommit

By default, MySQL runs in autocommit mode - each statement is its own transaction and is committed immediately:

```sql
SHOW VARIABLES LIKE 'autocommit';
```

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
```

When you issue `START TRANSACTION` or `BEGIN`, MySQL temporarily disables autocommit for that transaction scope.

## Disabling Autocommit

You can disable autocommit for the session, requiring explicit commits:

```sql
SET autocommit = 0;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- Not yet committed

COMMIT;  -- Now committed
```

Re-enable autocommit after:

```sql
SET autocommit = 1;
```

## START TRANSACTION Options

MySQL 8.0 supports options on `START TRANSACTION`:

```sql
-- Read-only transaction (optimization for read-heavy transactions)
START TRANSACTION READ ONLY;

-- Read-write transaction (default)
START TRANSACTION READ WRITE;

-- Consistent snapshot (sets the read view at transaction start)
START TRANSACTION WITH CONSISTENT SNAPSHOT;
```

`WITH CONSISTENT SNAPSHOT` starts a repeatable read view immediately - useful for long-running read operations that need consistent data throughout.

## Checking Transaction Status

```sql
-- Check if a transaction is active
SELECT @@in_transaction;
```

Returns `1` if inside a transaction, `0` otherwise.

## DDL and Transactions

Note that DDL statements (`ALTER TABLE`, `CREATE TABLE`, `DROP TABLE`) cause an implicit commit in MySQL. A transaction cannot span DDL:

```sql
START TRANSACTION;
INSERT INTO orders VALUES (...);  -- part of transaction

ALTER TABLE orders ADD COLUMN notes TEXT;  -- implicit COMMIT here!

-- The INSERT above is committed even if you meant to roll it back
```

## Summary

Start a MySQL transaction with `START TRANSACTION` or `BEGIN`. All subsequent DML statements are grouped together until `COMMIT` saves them permanently or `ROLLBACK` discards them. MySQL's default autocommit mode commits each statement individually - disable it or use explicit transaction blocks for multi-statement operations. Always use error handling in application code to ensure transactions are rolled back on failure.
