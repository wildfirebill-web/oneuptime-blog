# How to Handle Mixed Collation Errors in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Collation, Error, Mixed Collation, Join

Description: Learn how to diagnose and fix the "Illegal mix of collations" error in MySQL when joining or comparing columns with different collations.

---

## Understanding the Error

When you compare or join string columns that have different collations, MySQL cannot determine which collation rules to apply and raises:

```text
ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT)
and (utf8mb4_general_ci,IMPLICIT) for operation '='
```

This commonly occurs when:
- Two tables were created with different default collations.
- A column has an explicit collation that differs from the one used in a query parameter.
- The connection character set differs from the column collation.

## Diagnosing the Problem

Check the collations of the columns involved in the failing query:

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME IN ('orders', 'customers')
  AND COLUMN_NAME IN ('customer_id', 'id');
```

Also check your session collation:

```sql
SHOW VARIABLES LIKE 'collation_connection';
```

## Fix 1 - Use COLLATE in the Query

Apply a matching collation to one side of the comparison to make them compatible:

```sql
SELECT o.id, c.name
FROM orders o
JOIN customers c
    ON o.customer_ref COLLATE utf8mb4_unicode_ci = c.id;
```

This is a quick fix but can prevent index use if the column's native collation differs.

## Fix 2 - Alter the Column to Match

The permanent solution is to align the collations of related columns:

```sql
-- Check current collation
SHOW CREATE TABLE orders\G

-- Change the column collation
ALTER TABLE orders
    MODIFY COLUMN customer_ref VARCHAR(50)
        CHARACTER SET utf8mb4
        COLLATE utf8mb4_unicode_ci;
```

After the change, rerun the query without any `COLLATE` override.

## Fix 3 - Standardize at the Database Level

Prevent future mismatch by setting the database default before creating any tables:

```sql
ALTER DATABASE your_database
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

New tables created without explicit collation will inherit this default.

## Fix 4 - Use CONVERT() to Coerce at Query Time

```sql
SELECT o.id, c.name
FROM orders o
JOIN customers c
    ON CONVERT(o.customer_ref USING utf8mb4) COLLATE utf8mb4_unicode_ci = c.id;
```

## Generating a Migration Script

If many columns are mismatched, generate ALTER statements from `information_schema`:

```sql
SELECT CONCAT(
    'ALTER TABLE `', TABLE_NAME, '` MODIFY COLUMN `', COLUMN_NAME, '` ',
    COLUMN_TYPE, ' CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;'
) AS fix_statement
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA   = 'your_database'
  AND COLLATION_NAME != 'utf8mb4_unicode_ci'
  AND DATA_TYPE IN ('varchar', 'char', 'text', 'tinytext', 'mediumtext', 'longtext');
```

Review and run each statement individually.

## Summary

Mixed collation errors arise from schema inconsistency. For a quick fix, use the `COLLATE` clause inline; for a lasting solution, align all related columns to the same character set and collation using `ALTER TABLE`. Setting a consistent database-level default prevents the problem from recurring in new tables.
