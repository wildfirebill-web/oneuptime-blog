# How to Use TRUNCATE TABLE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Truncate Table, Ddl, Sql, Database

Description: Learn how to use TRUNCATE TABLE in MySQL to quickly delete all rows from a table while resetting auto-increment counters, and how it differs from DELETE.

---

## Introduction

`TRUNCATE TABLE` is a DDL (Data Definition Language) statement in MySQL that quickly removes all rows from a table. It is faster than `DELETE FROM table` because it drops and recreates the table structure internally rather than deleting rows one by one. It also resets any auto-increment counters to their starting value.

## Basic Syntax

```sql
TRUNCATE TABLE table_name;
```

Or simply:

```sql
TRUNCATE table_name;
```

## Simple Example

```sql
-- Remove all rows from the logs table
TRUNCATE TABLE logs;
```

After this, the `logs` table exists but is empty, and the auto-increment counter is reset to 1.

## TRUNCATE vs DELETE

| Feature | TRUNCATE | DELETE |
|---------|----------|--------|
| Removes all rows | Yes | Yes (without WHERE) |
| WHERE clause support | No | Yes |
| Speed | Fast (DDL operation) | Slower (DML, row by row) |
| Resets AUTO_INCREMENT | Yes | No |
| Can be rolled back | No (implicit commit) | Yes (in transaction) |
| Triggers fired | No | Yes |
| Returns rows deleted | No | Yes (ROW_COUNT()) |
| Foreign key checks | Yes (may fail) | Yes |

## TRUNCATE vs DROP TABLE

- `TRUNCATE TABLE`: removes all rows, keeps the table structure (columns, indexes, constraints).
- `DROP TABLE`: removes the entire table including its definition.

```sql
-- Keep the table, clear the data
TRUNCATE TABLE session_cache;

-- Remove the table entirely
DROP TABLE session_cache;
```

## Resetting AUTO_INCREMENT

One of the key behaviors of TRUNCATE is resetting the auto-increment counter:

```sql
INSERT INTO items (name) VALUES ('A'), ('B'), ('C');
-- items now has IDs 1, 2, 3

TRUNCATE TABLE items;

INSERT INTO items (name) VALUES ('X');
-- items now has ID 1 (counter reset)
```

With `DELETE`, the counter is not reset:

```sql
DELETE FROM items;
INSERT INTO items (name) VALUES ('X');
-- items now has ID 4 (counter continues from last value)
```

## TRUNCATE and Foreign Keys

If a table has rows referenced by a foreign key in another table, `TRUNCATE` will fail:

```sql
-- orders references customers via customer_id
TRUNCATE TABLE customers;
-- ERROR: Cannot truncate a table referenced in a foreign key constraint
```

To truncate both, disable foreign key checks temporarily:

```sql
SET foreign_key_checks = 0;
TRUNCATE TABLE orders;
TRUNCATE TABLE customers;
SET foreign_key_checks = 1;
```

## TRUNCATE Does Not Fire Triggers

Unlike `DELETE`, `TRUNCATE` does not activate `DELETE` triggers on the table.

```sql
-- This trigger will NOT fire during TRUNCATE
CREATE TRIGGER after_delete_log
AFTER DELETE ON orders
FOR EACH ROW
INSERT INTO audit_log (action, order_id) VALUES ('deleted', OLD.id);

-- TRUNCATE bypasses triggers
TRUNCATE TABLE orders;
```

## TRUNCATE Cannot Be Used in a Transaction

In MySQL, `TRUNCATE` causes an implicit commit. You cannot roll it back:

```sql
START TRANSACTION;
TRUNCATE TABLE logs; -- implicit commit happens here
ROLLBACK;            -- does nothing, TRUNCATE already committed
```

## Partitioned Table Truncation

Truncate specific partitions without removing the whole table:

```sql
ALTER TABLE logs TRUNCATE PARTITION p2023;
```

Or truncate all partitions:

```sql
TRUNCATE TABLE logs; -- truncates all partitions
```

## Practical Use Cases

- Clearing test data after automated tests.
- Resetting staging tables before a new data load.
- Emptying log or cache tables during maintenance.
- Resetting sequence counters in development environments.

```sql
-- After each test suite run, reset the test database tables
TRUNCATE TABLE test_users;
TRUNCATE TABLE test_orders;
TRUNCATE TABLE test_audit_log;
```

## Summary

`TRUNCATE TABLE` is the fastest way to delete all rows from a MySQL table. It resets auto-increment counters, cannot be rolled back, does not fire triggers, and may fail if foreign key constraints are present. Use it for clearing test data, resetting caches, and bulk data refreshes where the table structure should be preserved.
