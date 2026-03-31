# How to Transpose Rows to Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Pivot

Description: Learn how to transpose rows to columns in MySQL using CASE expressions and conditional aggregation, with static and dynamic pivot query techniques.

---

Transposing rows to columns, often called pivoting, is a common reporting task where values stored in rows need to appear as column headers. MySQL does not have a native `PIVOT` keyword like SQL Server, but you can achieve the same result with conditional aggregation and dynamic SQL.

## Static Pivot with CASE

For a known, fixed set of values, use `CASE` inside aggregate functions grouped by the row identifier:

```sql
SELECT
  student_id,
  MAX(CASE WHEN subject = 'Math'    THEN score END) AS Math,
  MAX(CASE WHEN subject = 'Science' THEN score END) AS Science,
  MAX(CASE WHEN subject = 'English' THEN score END) AS English
FROM grades
GROUP BY student_id;
```

Each column captures the score for a specific subject. `MAX()` is used because there is only one score per student per subject, so it acts as a selector rather than an actual aggregation.

## Using SUM Instead of MAX

When values need to be accumulated rather than selected, switch to `SUM()`:

```sql
SELECT
  salesperson_id,
  SUM(CASE WHEN QUARTER(sale_date) = 1 THEN amount ELSE 0 END) AS Q1,
  SUM(CASE WHEN QUARTER(sale_date) = 2 THEN amount ELSE 0 END) AS Q2,
  SUM(CASE WHEN QUARTER(sale_date) = 3 THEN amount ELSE 0 END) AS Q3,
  SUM(CASE WHEN QUARTER(sale_date) = 4 THEN amount ELSE 0 END) AS Q4
FROM sales
GROUP BY salesperson_id;
```

This produces a quarterly breakdown per salesperson from a normalized transaction table.

## Dynamic Pivot Using Prepared Statements

When the set of values is not known in advance, generate the column list dynamically using `GROUP_CONCAT()` and execute it as a prepared statement:

```sql
SET @columns = NULL;

SELECT GROUP_CONCAT(
  DISTINCT CONCAT(
    'MAX(CASE WHEN subject = ''', subject, ''' THEN score END) AS `', subject, '`'
  )
  ORDER BY subject
) INTO @columns
FROM grades;

SET @query = CONCAT(
  'SELECT student_id, ', @columns,
  ' FROM grades GROUP BY student_id'
);

PREPARE stmt FROM @query;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;
```

This approach adapts automatically to new subject values without modifying the query.

## Transposing Multiple Metrics

You can pivot multiple metrics simultaneously by expanding the column list:

```sql
SELECT
  department,
  SUM(CASE WHEN status = 'active'   THEN salary END) AS active_payroll,
  COUNT(CASE WHEN status = 'active'   THEN 1    END) AS active_headcount,
  SUM(CASE WHEN status = 'inactive' THEN salary END) AS inactive_payroll,
  COUNT(CASE WHEN status = 'inactive' THEN 1    END) AS inactive_headcount
FROM employees
GROUP BY department;
```

## Handling NULL Values

Pivoted columns produce `NULL` when no matching row exists. Replace them with a default using `COALESCE()`:

```sql
SELECT
  student_id,
  COALESCE(MAX(CASE WHEN subject = 'Math'    THEN score END), 0) AS Math,
  COALESCE(MAX(CASE WHEN subject = 'Science' THEN score END), 0) AS Science
FROM grades
GROUP BY student_id;
```

## Performance Tips

Static pivot queries benefit from indexes on the column being pivoted (the `subject` column in these examples) and on the grouping column (`student_id`). A composite index on `(student_id, subject, score)` can make the query entirely index-covered, avoiding table lookups.

## Summary

MySQL does not support native pivot syntax, but conditional aggregation with `CASE` expressions provides an equally powerful alternative. For static value sets, write the columns directly. For dynamic sets, use `GROUP_CONCAT()` to build and execute a prepared statement at runtime.
