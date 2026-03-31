# How to Reset AUTO_INCREMENT Value in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Database, Schema, Auto Increment, Table

Description: Learn how to reset or change the AUTO_INCREMENT counter in MySQL tables, why it matters, and important caveats to avoid duplicate key errors.

---

## Why Reset AUTO_INCREMENT

The `AUTO_INCREMENT` counter tracks the next integer value assigned when a new row is inserted without an explicit ID. You may want to reset it when:

- You deleted all rows and want IDs to start from 1 again.
- You are migrating data and need a clean sequence.
- The counter jumped significantly after bulk deletes and you want to reclaim gaps.

## Check the Current AUTO_INCREMENT Value

```sql
SHOW CREATE TABLE orders\G
-- Look for AUTO_INCREMENT=1042 in the output

-- Or query information_schema
SELECT AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';
```

## Reset AUTO_INCREMENT Using ALTER TABLE

```sql
-- Set the counter to a specific value
ALTER TABLE orders AUTO_INCREMENT = 1;

-- Or any value you want the next insert to use
ALTER TABLE orders AUTO_INCREMENT = 100;
```

Important: MySQL will silently raise this value to `MAX(id) + 1` if the value you specify is less than or equal to the current maximum. You cannot set `AUTO_INCREMENT` below the highest existing ID.

## Safe Reset After Deleting All Rows

```sql
-- First confirm the table is empty
SELECT COUNT(*) FROM orders;

-- Then reset
TRUNCATE TABLE orders;
-- TRUNCATE automatically resets AUTO_INCREMENT to 1
```

```sql
-- Alternative: DELETE + ALTER TABLE
DELETE FROM orders;
ALTER TABLE orders AUTO_INCREMENT = 1;
```

`TRUNCATE` is faster and atomically resets the counter. `DELETE` preserves transaction history and triggers.

## Reset to the Minimum Safe Value

When rows still exist, you cannot go below the current maximum. To reclaim gaps, set to `MAX(id) + 1`:

```sql
SET @max_id = (SELECT MAX(id) FROM orders);
SET @next_val = @max_id + 1;

-- Build and execute the statement dynamically
SET @sql = CONCAT('ALTER TABLE orders AUTO_INCREMENT = ', @next_val);
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Verify the Change

```sql
SELECT AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'orders';
```

## InnoDB Behavior and Persistence

In InnoDB (MySQL 8+), the `AUTO_INCREMENT` counter is persisted to the redo log, so it survives server restarts. In MySQL 5.7 and earlier, InnoDB re-derives the counter from `MAX(id)` at startup, which meant a restarted server could reuse IDs after deletes.

```sql
-- Check your MySQL version
SELECT VERSION();
```

## AUTO_INCREMENT Gaps Are Normal

Gaps in `AUTO_INCREMENT` sequences are expected and normal. They arise from:
- Rolled-back transactions
- `INSERT IGNORE` failures
- Bulk deletes

Do not design application logic that relies on sequential, gap-free IDs.

## Summary

You can reset the `AUTO_INCREMENT` counter with `ALTER TABLE t AUTO_INCREMENT = n`, but MySQL enforces that the new value must exceed the current maximum ID in the table. `TRUNCATE TABLE` is the most straightforward way to reset a counter to 1 when the table is empty. MySQL 8+ persists the counter across restarts, eliminating the ID-reuse issue present in older InnoDB versions.
