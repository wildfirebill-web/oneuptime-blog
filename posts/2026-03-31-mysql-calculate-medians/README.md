# How to Calculate Medians in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Statistics

Description: Learn multiple techniques to calculate medians in MySQL, including the classic row-counting method and the modern window function approach available in MySQL 8.0+.

---

MySQL does not have a built-in `MEDIAN()` function, unlike some other databases. Calculating a median requires finding the middle value in a sorted dataset, which takes a bit more work in SQL. There are several approaches, each with different trade-offs in readability and performance.

## What Is a Median

The median is the value that separates the higher half from the lower half of a dataset. For an odd number of rows it is the middle value; for an even number it is the average of the two middle values.

## Classic Method Using Subqueries

The traditional approach counts rows and selects the middle position:

```sql
SELECT AVG(salary) AS median_salary
FROM (
  SELECT salary
  FROM employees
  ORDER BY salary
  LIMIT 2 - (SELECT COUNT(*) FROM employees) MOD 2
  OFFSET (SELECT (COUNT(*) - 1) / 2 FROM employees)
) AS middle_rows;
```

The `LIMIT` expression evaluates to 1 for odd counts and 2 for even counts. `OFFSET` skips to the midpoint. `AVG()` on 1 or 2 rows returns the median correctly in both cases.

## Using Variables (MySQL 5.x Compatible)

For older MySQL versions without window functions:

```sql
SET @rownum = 0;
SET @total = (SELECT COUNT(*) FROM employees);

SELECT AVG(salary) AS median_salary
FROM (
  SELECT salary, @rownum := @rownum + 1 AS rn
  FROM employees
  ORDER BY salary
) ranked
WHERE rn IN (
  FLOOR((@total + 1) / 2),
  CEIL((@total + 1) / 2)
);
```

## Window Function Method (MySQL 8.0+)

MySQL 8.0 introduced `ROW_NUMBER()` and `COUNT()` as window functions, enabling a cleaner median calculation:

```sql
WITH ranked AS (
  SELECT
    salary,
    ROW_NUMBER() OVER (ORDER BY salary) AS rn,
    COUNT(*) OVER () AS total
  FROM employees
)
SELECT AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((total + 1) / 2), CEIL((total + 1) / 2));
```

This CTE-based approach is more readable and avoids session variables.

## Grouped Median Per Category

To compute medians for each group (such as median salary per department), use a partitioned window:

```sql
WITH ranked AS (
  SELECT
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
    COUNT(*) OVER (PARTITION BY department) AS dept_count
  FROM employees
)
SELECT
  department,
  AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (
  FLOOR((dept_count + 1) / 2),
  CEIL((dept_count + 1) / 2)
)
GROUP BY department;
```

## Median with Percentile Approximation

For large datasets where approximate results are acceptable, use `PERCENTILE_CONT` equivalents via sorting:

```sql
WITH ordered AS (
  SELECT
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
  FROM employees
)
SELECT
  AVG(salary) AS approx_median
FROM ordered
WHERE quartile = 2;
```

This is not exact but can be faster on very large tables.

## Performance Considerations

Median calculations typically require a full table sort. Ensure the column being sorted is indexed. For repeated median calculations, consider precomputing and caching the result, or maintaining a summary table via triggers.

```sql
-- Add index to improve sort performance
CREATE INDEX idx_employees_salary ON employees(salary);
```

## Summary

MySQL lacks a native `MEDIAN()` aggregate, but you can compute it using subquery offset tricks or, in MySQL 8.0+, with window functions and CTEs. The window function approach is the most readable and scales well to per-group medians using `PARTITION BY`.
