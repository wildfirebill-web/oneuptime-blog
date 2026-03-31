# How to Execute Dynamic Pivot Queries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Pivot, Query

Description: Learn how to execute dynamic pivot queries in MySQL using GROUP_CONCAT and prepared statements to transpose row values into columns without hardcoding them.

---

A dynamic pivot query generates column names from row data at runtime. Unlike static pivot queries where columns are hardcoded, dynamic pivots adapt to whatever distinct values exist in the database at query time. MySQL achieves this by building the column list with `GROUP_CONCAT` and executing the result as a prepared statement.

## The Challenge with Static Pivots

Static pivot queries hardcode each column:

```sql
SELECT
  salesperson,
  SUM(CASE WHEN product = 'Widget' THEN revenue ELSE 0 END) AS Widget,
  SUM(CASE WHEN product = 'Gadget' THEN revenue ELSE 0 END) AS Gadget
FROM sales
GROUP BY salesperson;
```

This breaks as soon as a new product is added. Dynamic pivots solve this by discovering the values automatically.

## Step 1: Discover the Distinct Values

```sql
SELECT DISTINCT product
FROM sales
ORDER BY product;
```

This shows all current product values that will become column headers.

## Step 2: Build the Column Expression List

Use `GROUP_CONCAT` to generate the `CASE` expressions as a single string:

```sql
SELECT GROUP_CONCAT(
  DISTINCT CONCAT(
    'SUM(CASE WHEN product = ''', product, ''' THEN revenue ELSE 0 END) AS `', product, '`'
  )
  ORDER BY product
  SEPARATOR ', '
) INTO @cols
FROM sales;
```

The double single-quotes (`''`) escape the single quotes within the string. Backtick quoting around the alias handles product names with spaces or special characters.

## Step 3: Build and Execute the Full Query

```sql
SET @query = CONCAT(
  'SELECT salesperson, ', @cols,
  ' FROM sales GROUP BY salesperson ORDER BY salesperson'
);

PREPARE stmt FROM @query;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Full Working Example

Here is the complete sequence in one block:

```sql
-- Generate column list
SELECT GROUP_CONCAT(
  DISTINCT CONCAT(
    'SUM(CASE WHEN product = ''', product, ''' THEN revenue ELSE 0 END) AS `', product, '`'
  )
  ORDER BY product
) INTO @cols
FROM sales;

-- Execute dynamic pivot
SET @sql = CONCAT('SELECT salesperson, ', @cols, ' FROM sales GROUP BY salesperson');
PREPARE stmt FROM @sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

## Adding Row Totals

Append a total column by including an unconditional `SUM`:

```sql
SET @sql = CONCAT(
  'SELECT salesperson, ',
  @cols,
  ', SUM(revenue) AS Total FROM sales GROUP BY salesperson'
);
```

## Wrapping in a Stored Procedure

For reuse, encapsulate the logic in a stored procedure:

```sql
DELIMITER //

CREATE PROCEDURE pivot_sales_by_product()
BEGIN
  SET @cols = NULL;

  SELECT GROUP_CONCAT(
    DISTINCT CONCAT(
      'SUM(CASE WHEN product = ''', product, ''' THEN revenue ELSE 0 END) AS `', product, '`'
    )
    ORDER BY product
  ) INTO @cols
  FROM sales;

  SET @sql = CONCAT(
    'SELECT salesperson, ', @cols, ', SUM(revenue) AS Total ',
    'FROM sales GROUP BY salesperson'
  );

  PREPARE stmt FROM @sql;
  EXECUTE stmt;
  DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL pivot_sales_by_product();
```

## Limitations

`GROUP_CONCAT` has a default maximum length of 1024 characters. For wide pivots, increase it:

```sql
SET SESSION group_concat_max_len = 1000000;
```

Also note that dynamic pivot results cannot be used as the base for further SQL transformations within the same query since the column set is not known at parse time.

## Summary

Dynamic pivot queries in MySQL use `GROUP_CONCAT` to generate `CASE` expression strings from distinct row values, then execute them via prepared statements. This produces a pivot table that automatically adjusts to new values, making it suitable for reporting on dimensions that evolve over time.
