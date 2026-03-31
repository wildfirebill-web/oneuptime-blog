# How to Use Pivot Tables (Rows to Columns) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pivot, Query, Aggregation, Reporting

Description: Transform MySQL row data into columns using conditional aggregation and dynamic SQL to produce pivot-style reports without native PIVOT syntax.

---

## Why Pivot in MySQL?

MySQL does not have a native `PIVOT` keyword like SQL Server or Oracle. Instead, you achieve the same result using conditional aggregation - wrapping aggregate functions inside `CASE` expressions. This technique works for both static pivots (known columns) and dynamic pivots (columns determined at runtime).

## Static Pivot with Conditional Aggregation

Suppose you have a sales table:

```sql
CREATE TABLE sales (
  id INT AUTO_INCREMENT PRIMARY KEY,
  salesperson VARCHAR(50),
  quarter CHAR(2),
  amount DECIMAL(10,2)
);

INSERT INTO sales (salesperson, quarter, amount) VALUES
('Alice', 'Q1', 5000), ('Alice', 'Q2', 7000),
('Bob',   'Q1', 4500), ('Bob',   'Q3', 6000),
('Alice', 'Q3', 8000), ('Bob',   'Q4', 3000);
```

Pivot to show each quarter as a column:

```sql
SELECT
  salesperson,
  SUM(CASE WHEN quarter = 'Q1' THEN amount ELSE 0 END) AS Q1,
  SUM(CASE WHEN quarter = 'Q2' THEN amount ELSE 0 END) AS Q2,
  SUM(CASE WHEN quarter = 'Q3' THEN amount ELSE 0 END) AS Q3,
  SUM(CASE WHEN quarter = 'Q4' THEN amount ELSE 0 END) AS Q4,
  SUM(amount) AS Total
FROM sales
GROUP BY salesperson
ORDER BY salesperson;
```

Result:

```text
+-----------+------+------+------+------+-------+
| salesperson | Q1  | Q2   | Q3   | Q4   | Total |
+-----------+------+------+------+------+-------+
| Alice     | 5000 | 7000 | 8000 |    0 | 20000 |
| Bob       | 4500 |    0 | 6000 | 3000 | 13500 |
+-----------+------+------+------+------+-------+
```

## Using IF Instead of CASE

`IF` is a shorter MySQL alternative:

```sql
SELECT
  salesperson,
  SUM(IF(quarter = 'Q1', amount, 0)) AS Q1,
  SUM(IF(quarter = 'Q2', amount, 0)) AS Q2,
  SUM(IF(quarter = 'Q3', amount, 0)) AS Q3,
  SUM(IF(quarter = 'Q4', amount, 0)) AS Q4
FROM sales
GROUP BY salesperson;
```

## Dynamic Pivot with Prepared Statements

When the pivot columns are not known in advance, build the query dynamically:

```sql
SET SESSION group_concat_max_len = 1000000;

SET @columns = NULL;
SELECT GROUP_CONCAT(
  DISTINCT CONCAT(
    "SUM(IF(quarter = '", quarter, "', amount, 0)) AS `", quarter, "`"
  )
  ORDER BY quarter
) INTO @columns
FROM sales;

SET @query = CONCAT(
  'SELECT salesperson, ', @columns,
  ' FROM sales GROUP BY salesperson ORDER BY salesperson'
);

PREPARE stmt FROM @query;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

This dynamically generates column names based on the actual values in the `quarter` column.

## Counting Instead of Summing

For count-based pivots:

```sql
SELECT
  salesperson,
  COUNT(IF(quarter = 'Q1', 1, NULL)) AS Q1_orders,
  COUNT(IF(quarter = 'Q2', 1, NULL)) AS Q2_orders
FROM sales
GROUP BY salesperson;
```

## Multi-Column Pivot

Pivot on two dimensions simultaneously:

```sql
SELECT
  product_category,
  SUM(IF(region = 'North' AND quarter = 'Q1', revenue, 0)) AS North_Q1,
  SUM(IF(region = 'South' AND quarter = 'Q1', revenue, 0)) AS South_Q1,
  SUM(IF(region = 'North' AND quarter = 'Q2', revenue, 0)) AS North_Q2,
  SUM(IF(region = 'South' AND quarter = 'Q2', revenue, 0)) AS South_Q2
FROM regional_sales
GROUP BY product_category;
```

## Summary

MySQL pivot tables use conditional aggregation with `CASE WHEN` or `IF` wrapped in aggregate functions like `SUM` or `COUNT`. For static pivots with known column values, write the `CASE` expressions directly. For dynamic pivots where column values change, use `GROUP_CONCAT` to build a prepared statement at runtime.
