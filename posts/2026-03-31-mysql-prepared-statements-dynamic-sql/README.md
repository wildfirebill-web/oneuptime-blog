# How to Use Prepared Statements for Dynamic SQL in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Prepared Statement, Dynamic SQL, Stored Procedure, Query

Description: Learn how to build and execute dynamic SQL queries in MySQL stored procedures using PREPARE and EXECUTE for runtime-constructed statements.

---

## What is Dynamic SQL?

Dynamic SQL refers to SQL statements that are constructed at runtime rather than hard-coded. MySQL stored procedures can build query strings in variables and execute them using `PREPARE` and `EXECUTE`. This enables:

- Queries against tables whose names are only known at runtime
- Conditionally added `WHERE` clauses
- Dynamic `ORDER BY` or `LIMIT` clauses
- Generic reporting procedures that work across multiple tables

## Basic Pattern

```sql
DELIMITER //
CREATE PROCEDURE query_any_table(IN tbl_name VARCHAR(64))
BEGIN
  SET @sql = CONCAT('SELECT COUNT(*) AS row_count FROM ', tbl_name);
  PREPARE stmt FROM @sql;
  EXECUTE stmt;
  DEALLOCATE PREPARE stmt;
END//
DELIMITER ;

CALL query_any_table('orders');
CALL query_any_table('products');
```

## Dynamic WHERE Clause

Build filter conditions based on which parameters are provided:

```sql
DELIMITER //
CREATE PROCEDURE search_products(
  IN p_category VARCHAR(50),
  IN p_max_price DECIMAL(10,2)
)
BEGIN
  SET @sql = 'SELECT id, name, price FROM products WHERE 1=1';

  IF p_category IS NOT NULL THEN
    SET @sql = CONCAT(@sql, ' AND category = ?');
    SET @cat = p_category;
  END IF;

  IF p_max_price IS NOT NULL THEN
    SET @sql = CONCAT(@sql, ' AND price <= ?');
    SET @price = p_max_price;
  END IF;

  PREPARE stmt FROM @sql;

  IF p_category IS NOT NULL AND p_max_price IS NOT NULL THEN
    EXECUTE stmt USING @cat, @price;
  ELSEIF p_category IS NOT NULL THEN
    EXECUTE stmt USING @cat;
  ELSEIF p_max_price IS NOT NULL THEN
    EXECUTE stmt USING @price;
  ELSE
    EXECUTE stmt;
  END IF;

  DEALLOCATE PREPARE stmt;
END//
DELIMITER ;

CALL search_products('electronics', 299.99);
CALL search_products(NULL, 100.00);
```

## Dynamic ORDER BY

Column names cannot be parameterized with `?`, so dynamic sort columns must be concatenated into the query string. Always validate the column name against an allowed list to prevent injection:

```sql
DELIMITER //
CREATE PROCEDURE list_products_sorted(IN sort_col VARCHAR(30))
BEGIN
  -- Whitelist allowed sort columns
  IF sort_col NOT IN ('name', 'price', 'created_at') THEN
    SET sort_col = 'name';
  END IF;

  SET @sql = CONCAT(
    'SELECT id, name, price FROM products ORDER BY ', sort_col
  );

  PREPARE stmt FROM @sql;
  EXECUTE stmt;
  DEALLOCATE PREPARE stmt;
END//
DELIMITER ;

CALL list_products_sorted('price');
```

## Dynamic LIMIT and OFFSET for Pagination

```sql
DELIMITER //
CREATE PROCEDURE paginate_orders(IN page INT, IN page_size INT)
BEGIN
  SET @offset = (page - 1) * page_size;
  SET @limit = page_size;
  SET @sql = 'SELECT * FROM orders ORDER BY id LIMIT ? OFFSET ?';

  PREPARE stmt FROM @sql;
  EXECUTE stmt USING @limit, @offset;
  DEALLOCATE PREPARE stmt;
END//
DELIMITER ;

CALL paginate_orders(2, 20);
```

## Inspecting the Dynamic Query

Before executing, log or output the constructed SQL for debugging:

```sql
SET @sql = CONCAT('SELECT * FROM ', @table_name, ' WHERE id = ?');
SELECT @sql AS constructed_query;
PREPARE stmt FROM @sql;
```

## Limitations

- Table names and column names cannot use `?` parameters - they must be concatenated.
- The `PREPARE` statement cannot be nested inside another `PREPARE`.
- Prepared statements only exist within the current session.

## Summary

Dynamic SQL with `PREPARE` and `EXECUTE` enables stored procedures to build and run queries at runtime. Use it for generic query patterns, runtime table selection, and conditional filtering. Always validate any user-supplied structural elements (table names, column names) against an allowed list before concatenating them into the query string to prevent SQL injection through dynamic SQL construction.
