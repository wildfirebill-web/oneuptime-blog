# How to Insert a Single Row in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert, DML, Data Manipulation, Query

Description: Learn the syntax and best practices for inserting a single row into a MySQL table using the INSERT INTO statement.

---

## Basic INSERT INTO Syntax

The `INSERT INTO` statement adds a new row to a table. You specify the table name, the column names, and the values to insert:

```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

Always list column names explicitly. This protects your query from breaking if columns are added or reordered later.

## Simple Row Insert

Insert a new user into a users table:

```sql
INSERT INTO users (username, email, created_at)
VALUES ('alice', 'alice@example.com', NOW());
```

## Using DEFAULT Values

If a column has a default value, you can omit it from the column list or use the `DEFAULT` keyword:

```sql
-- Omit the column entirely
INSERT INTO orders (user_id, amount) VALUES (42, 150.00);

-- Or use DEFAULT explicitly
INSERT INTO orders (user_id, amount, status)
VALUES (42, 150.00, DEFAULT);
```

## Retrieving the Auto-Increment ID

After inserting a row with an auto-increment primary key, retrieve the generated ID using `LAST_INSERT_ID()`:

```sql
INSERT INTO products (name, price, category)
VALUES ('Wireless Mouse', 29.99, 'Electronics');

SELECT LAST_INSERT_ID();
```

In application code, most drivers expose this automatically:

```python
cursor.execute(
    "INSERT INTO products (name, price, category) VALUES (%s, %s, %s)",
    ('Wireless Mouse', 29.99, 'Electronics')
)
new_id = cursor.lastrowid
```

## Inserting NULL Values

Use `NULL` explicitly for nullable columns:

```sql
INSERT INTO employees (name, department, manager_id)
VALUES ('Bob', 'Engineering', NULL);
```

## Inserting with Expressions

Column values can be expressions, not just literals:

```sql
INSERT INTO audit_log (event, user_id, occurred_at)
VALUES ('login', 7, UTC_TIMESTAMP());
```

```sql
INSERT INTO pricing (product_id, base_price, discounted_price)
VALUES (101, 100.00, 100.00 * 0.85);
```

## Verifying the Insert

Confirm the row was added:

```sql
SELECT * FROM users WHERE username = 'alice';
```

Or check the row count:

```sql
SELECT ROW_COUNT();  -- Returns 1 after a successful single-row insert
```

## Common Errors

- **Duplicate entry**: The primary key or unique index value already exists. Use `INSERT IGNORE` or `INSERT ... ON DUPLICATE KEY UPDATE` to handle this gracefully.
- **Data too long**: The value exceeds the column size. Check column definitions with `DESCRIBE table_name`.
- **Cannot be null**: A NOT NULL column was not given a value. Add the column to the INSERT or provide a default.

## Summary

Inserting a single row in MySQL requires specifying the target table, the columns to populate, and the corresponding values. Always list column names explicitly, use `LAST_INSERT_ID()` to retrieve auto-generated keys, and handle potential constraint violations with `INSERT IGNORE` or `ON DUPLICATE KEY UPDATE` as needed.
