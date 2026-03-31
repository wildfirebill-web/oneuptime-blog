# How to Use CREATE TABLE ... SELECT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Create Table Select, Table Copy, Sql, Database

Description: Learn how to use CREATE TABLE ... SELECT in MySQL to create a new table and populate it with data from an existing table in a single statement.

---

## Introduction

`CREATE TABLE ... SELECT` is a MySQL shorthand that creates a new table and populates it with data from a SELECT query in one statement. It is useful for quickly creating table copies, snapshots, or derived tables based on existing data.

## Basic Syntax

```sql
CREATE TABLE new_table AS
SELECT column1, column2, ...
FROM existing_table
WHERE condition;
```

Or the alternative form:

```sql
CREATE TABLE new_table
SELECT column1, column2, ...
FROM existing_table;
```

## Simple Table Copy

Create a complete copy of an existing table's data:

```sql
CREATE TABLE employees_backup AS
SELECT * FROM employees;
```

This creates `employees_backup` with the same columns and data as `employees`.

## Selective Column Copy

Copy only specific columns:

```sql
CREATE TABLE employee_contacts AS
SELECT id, name, email, phone
FROM employees;
```

## Copy with Filtering

Create a table containing only a subset of rows:

```sql
CREATE TABLE active_customers AS
SELECT id, name, email, tier
FROM customers
WHERE status = 'active'
  AND last_order_date >= DATE_SUB(NOW(), INTERVAL 1 YEAR);
```

## Creating a Summary Table

Create an aggregated table from existing data:

```sql
CREATE TABLE department_stats AS
SELECT
  department,
  COUNT(*) AS employee_count,
  AVG(salary) AS avg_salary,
  MIN(salary) AS min_salary,
  MAX(salary) AS max_salary
FROM employees
GROUP BY department;
```

## Adding Computed Columns

Include expressions and computed values:

```sql
CREATE TABLE order_summaries AS
SELECT
  o.id AS order_id,
  c.name AS customer_name,
  o.total_amount,
  o.total_amount * 0.1 AS tax_amount,
  o.total_amount * 1.1 AS total_with_tax,
  DATEDIFF(NOW(), o.order_date) AS days_ago
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

## IMPORTANT: Indexes Are NOT Copied

`CREATE TABLE ... SELECT` copies column definitions and data, but does NOT copy:
- Primary keys
- Indexes
- Foreign keys
- Auto-increment settings
- Default values (in some cases)

Add them manually after creation:

```sql
CREATE TABLE products_copy AS SELECT * FROM products;

-- Add primary key and index manually
ALTER TABLE products_copy ADD PRIMARY KEY (id);
ALTER TABLE products_copy ADD INDEX idx_sku (sku);
ALTER TABLE products_copy MODIFY id INT AUTO_INCREMENT;
```

## CREATE TABLE IF NOT EXISTS ... SELECT

Avoid errors if the table already exists:

```sql
CREATE TABLE IF NOT EXISTS temp_results AS
SELECT id, score FROM exam_results WHERE passed = 1;
```

## Combining with UNION ALL

Create a table from multiple source tables:

```sql
CREATE TABLE all_transactions AS
SELECT id, amount, transaction_date, 'debit' AS type FROM debit_transactions
UNION ALL
SELECT id, amount, transaction_date, 'credit' AS type FROM credit_transactions;
```

## Creating an Empty Table with Same Structure

To create a table with the same structure but no data, use a false WHERE condition:

```sql
CREATE TABLE orders_staging AS
SELECT * FROM orders WHERE 1 = 0;
```

This creates the table with all columns but zero rows.

## Differences from SELECT INTO OUTFILE

`CREATE TABLE ... SELECT` creates a physical table in the database. `SELECT INTO OUTFILE` exports data to a file. Use `CREATE TABLE ... SELECT` when you need a new queryable table.

## Example: Snapshot Before a Risky Update

Before running a major update, save the current state:

```sql
CREATE TABLE products_snapshot_20240101 AS
SELECT * FROM products;

-- Now safely run the update
UPDATE products SET price = price * 1.05;

-- If needed, restore from snapshot
INSERT INTO products SELECT * FROM products_snapshot_20240101;
```

## Performance Considerations

- For large tables, the SELECT runs first, then the table is created and populated. This can take significant time.
- The new table is not visible to other sessions until the operation completes.
- Consider running this during off-peak hours for production tables.

```sql
-- Use LIMIT to test on a small subset first
CREATE TABLE large_table_sample AS
SELECT * FROM large_table LIMIT 1000;
```

## Summary

`CREATE TABLE ... SELECT` is a convenient one-step command to create and populate a new MySQL table from a query. It supports filtering, transformation, and joining, but does not copy indexes or constraints - add those manually. It is ideal for snapshots, data backups, staging tables, and derived datasets.
