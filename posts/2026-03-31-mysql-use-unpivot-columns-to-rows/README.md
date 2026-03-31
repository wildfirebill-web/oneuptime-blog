# How to Use Unpivot (Columns to Rows) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Unpivot, Query, Transform, Reporting

Description: Transform wide MySQL tables with many columns into normalized row-based formats using UNION ALL, a technique called unpivot or column-to-row transformation.

---

## What Is Unpivot?

Unpivot is the reverse of pivot: it takes multiple columns and rotates them into rows. MySQL has no native `UNPIVOT` keyword, but `UNION ALL` achieves the same result. This is useful when you have denormalized data with one column per category and need to normalize it for analysis or storage.

## Example - Wide Sales Table

Suppose you have quarterly sales stored as columns:

```sql
CREATE TABLE quarterly_sales (
  salesperson VARCHAR(50),
  Q1 DECIMAL(10,2),
  Q2 DECIMAL(10,2),
  Q3 DECIMAL(10,2),
  Q4 DECIMAL(10,2)
);

INSERT INTO quarterly_sales VALUES
('Alice', 5000, 7000, 8000, 0),
('Bob',   4500, 0,    6000, 3000);
```

## Unpivot Using UNION ALL

```sql
SELECT salesperson, 'Q1' AS quarter, Q1 AS amount FROM quarterly_sales WHERE Q1 > 0
UNION ALL
SELECT salesperson, 'Q2' AS quarter, Q2 AS amount FROM quarterly_sales WHERE Q2 > 0
UNION ALL
SELECT salesperson, 'Q3' AS quarter, Q3 AS amount FROM quarterly_sales WHERE Q3 > 0
UNION ALL
SELECT salesperson, 'Q4' AS quarter, Q4 AS amount FROM quarterly_sales WHERE Q4 > 0
ORDER BY salesperson, quarter;
```

Result:

```text
+-------------+---------+--------+
| salesperson | quarter | amount |
+-------------+---------+--------+
| Alice       | Q1      |   5000 |
| Alice       | Q2      |   7000 |
| Alice       | Q3      |   8000 |
| Bob         | Q1      |   4500 |
| Bob         | Q3      |   6000 |
| Bob         | Q4      |   3000 |
+-------------+---------+--------+
```

## Including NULL Values

To include rows with NULL or zero amounts, remove the WHERE filter:

```sql
SELECT salesperson, 'Q1' AS quarter, Q1 AS amount FROM quarterly_sales
UNION ALL
SELECT salesperson, 'Q2' AS quarter, Q2 AS amount FROM quarterly_sales
UNION ALL
SELECT salesperson, 'Q3' AS quarter, Q3 AS amount FROM quarterly_sales
UNION ALL
SELECT salesperson, 'Q4' AS quarter, Q4 AS amount FROM quarterly_sales
ORDER BY salesperson, quarter;
```

## Unpivot Multiple Measure Columns

If each period has both a revenue and units column:

```sql
SELECT salesperson, 'Q1' AS quarter, q1_revenue AS revenue, q1_units AS units
FROM wide_sales
UNION ALL
SELECT salesperson, 'Q2', q2_revenue, q2_units FROM wide_sales
UNION ALL
SELECT salesperson, 'Q3', q3_revenue, q3_units FROM wide_sales
ORDER BY salesperson, quarter;
```

## Using a Numbers Table for Dynamic Unpivot

For tables with many columns, a helper approach avoids repetition by joining with a sequence:

```sql
-- Assuming a numbers table with values 1-4
SELECT
  s.salesperson,
  CONCAT('Q', n.n) AS quarter,
  CASE n.n
    WHEN 1 THEN s.Q1
    WHEN 2 THEN s.Q2
    WHEN 3 THEN s.Q3
    WHEN 4 THEN s.Q4
  END AS amount
FROM quarterly_sales s
CROSS JOIN (SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4) n
ORDER BY s.salesperson, n.n;
```

## Inserting Unpivoted Data into a Normalized Table

```sql
INSERT INTO sales (salesperson, quarter, amount)
SELECT salesperson, 'Q1', Q1 FROM quarterly_sales WHERE Q1 IS NOT NULL AND Q1 > 0
UNION ALL
SELECT salesperson, 'Q2', Q2 FROM quarterly_sales WHERE Q2 IS NOT NULL AND Q2 > 0
UNION ALL
SELECT salesperson, 'Q3', Q3 FROM quarterly_sales WHERE Q3 IS NOT NULL AND Q3 > 0
UNION ALL
SELECT salesperson, 'Q4', Q4 FROM quarterly_sales WHERE Q4 IS NOT NULL AND Q4 > 0;
```

## Summary

MySQL's UNION ALL is the standard tool for unpivoting - rotating column values into rows. Write one SELECT per column you want to unpivot, assigning a literal label to identify the original column, then union the results together. For wide tables with many columns, a CROSS JOIN with a small numbers table reduces repetition.
