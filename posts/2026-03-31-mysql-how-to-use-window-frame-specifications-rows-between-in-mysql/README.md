# How to Use Window Frame Specifications (ROWS BETWEEN) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Functions, Sql, Query Optimization

Description: Learn how to use ROWS BETWEEN window frame specifications in MySQL to define precise physical row boundaries for window function calculations.

---

## What Are Window Frame Specifications?

Window functions in MySQL operate over a "frame" - a subset of rows within the current partition. The `ROWS BETWEEN` clause defines this frame using physical row offsets from the current row. This gives you precise, row-by-row control over which rows are included in the calculation.

## Syntax

```sql
function() OVER (
    [PARTITION BY ...]
    ORDER BY column
    ROWS BETWEEN <start> AND <end>
)
```

Frame boundaries can be:

```text
UNBOUNDED PRECEDING   -- first row of the partition
N PRECEDING           -- N rows before the current row
CURRENT ROW           -- the current row itself
N FOLLOWING           -- N rows after the current row
UNBOUNDED FOLLOWING   -- last row of the partition
```

## Common Frame Patterns

```sql
-- Cumulative (running total): from partition start to current row
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Trailing window of 3 rows: current + 2 preceding
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW

-- Centered window: 1 before, current, 1 after
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING

-- Look-ahead: current + 2 following
ROWS BETWEEN CURRENT ROW AND 2 FOLLOWING

-- Entire partition
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

## Sample Data

```sql
CREATE TABLE daily_sales (
    sale_date  DATE,
    rep_name   VARCHAR(50),
    amount     DECIMAL(10,2)
);

INSERT INTO daily_sales VALUES
  ('2024-01-01', 'Alice', 200),
  ('2024-01-02', 'Alice', 300),
  ('2024-01-03', 'Alice', 150),
  ('2024-01-04', 'Alice', 400),
  ('2024-01-05', 'Alice', 250);
```

## Running Total (UNBOUNDED PRECEDING to CURRENT ROW)

```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_sales;
```

## 3-Row Trailing Average

```sql
SELECT
    sale_date,
    amount,
    ROUND(
        AVG(amount) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2
    ) AS trailing_avg_3
FROM daily_sales;
```

## Centered 3-Row Average

```sql
SELECT
    sale_date,
    amount,
    ROUND(
        AVG(amount) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
        ), 2
    ) AS centered_avg
FROM daily_sales;
```

## Maximum in the Entire Partition

```sql
SELECT
    sale_date,
    amount,
    MAX(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS partition_max
FROM daily_sales;
```

## Difference from RANGE BETWEEN

`ROWS` uses physical row positions. `RANGE` uses logical value ranges and groups tied values:

```sql
-- Insert tied dates to illustrate:
INSERT INTO daily_sales VALUES
  ('2024-01-05', 'Alice', 100);

-- ROWS: each physical row is counted individually
SUM(amount) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- RANGE: both rows on 2024-01-05 are in the same boundary
SUM(amount) OVER (ORDER BY sale_date RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

For numeric running totals without ties, both produce the same result. When ties exist, `ROWS` accumulates them one at a time while `RANGE` includes all tied rows simultaneously.

## Practical Example - Detecting Anomalies

Flag rows where the current amount is more than twice the average of the 3 preceding rows:

```sql
SELECT
    sale_date,
    amount,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
    ) AS prev_avg,
    CASE
        WHEN amount > 2 * AVG(amount) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING
        ) THEN 'Anomaly'
        ELSE 'Normal'
    END AS status
FROM daily_sales;
```

## Summary

`ROWS BETWEEN` window frame specifications let you define precise physical row ranges for MySQL window functions, from running totals (`UNBOUNDED PRECEDING AND CURRENT ROW`) to sliding windows (`N PRECEDING AND N FOLLOWING`). Unlike `RANGE BETWEEN`, `ROWS` counts each physical row individually, making it the preferred choice when tied values should not be treated as equivalent frame boundaries.
