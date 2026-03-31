# How to Use Window Frame Specifications (RANGE BETWEEN) in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, SQL, Query Optimization

Description: Learn how to use RANGE BETWEEN window frame specifications in MySQL to define logical value-based boundaries for window function calculations.

---

## What Is RANGE BETWEEN?

In MySQL window functions, `RANGE BETWEEN` defines a frame based on the logical value range of the `ORDER BY` column rather than physical row positions. Rows with the same `ORDER BY` value are treated as peers and included together within the same boundary.

## Syntax

```sql
function() OVER (
    [PARTITION BY ...]
    ORDER BY column
    RANGE BETWEEN <start> AND <end>
)
```

Frame boundary options:

```text
UNBOUNDED PRECEDING   -- from the start of the partition
CURRENT ROW           -- all rows with the same ORDER BY value as current
UNBOUNDED FOLLOWING   -- to the end of the partition
```

Note: `RANGE BETWEEN N PRECEDING AND CURRENT ROW` (numeric offsets) requires a numeric or date `ORDER BY` column in MySQL 8.0.17+.

## Default Behavior

When you write `ORDER BY` without a frame clause, MySQL uses:

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

This is the implicit default - so cumulative aggregates by default use `RANGE`, not `ROWS`.

## Sample Data

```sql
CREATE TABLE orders (
    order_date DATE,
    customer   VARCHAR(50),
    total      DECIMAL(10,2)
);

INSERT INTO orders VALUES
  ('2024-01-01', 'A', 100),
  ('2024-01-02', 'A', 200),
  ('2024-01-02', 'A', 150),  -- same date as previous
  ('2024-01-03', 'A', 300),
  ('2024-01-04', 'A', 250);
```

## RANGE vs ROWS with Tied Values

```sql
SELECT
    order_date,
    total,
    SUM(total) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS rows_cumsum,
    SUM(total) OVER (
        ORDER BY order_date
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS range_cumsum
FROM orders
ORDER BY order_date, total;
```

Output:

```text
order_date  | total  | rows_cumsum | range_cumsum
------------+--------+-------------+--------------
2024-01-01  | 100.00 |      100.00 |       100.00
2024-01-02  | 150.00 |      250.00 |       450.00  (includes all 2024-01-02 rows)
2024-01-02  | 200.00 |      450.00 |       450.00  (same boundary)
2024-01-03  | 300.00 |      750.00 |       750.00
2024-01-04  | 250.00 |     1000.00 |      1000.00
```

With `RANGE`, both rows on `2024-01-02` are peers, so both receive the same cumulative sum that includes all rows up through that date.

## RANGE with Numeric Offsets

In MySQL 8.0.17+, you can specify numeric or interval offsets for `RANGE`:

```sql
-- Include all rows within 1 day before and 1 day after the current row's date
SELECT
    order_date,
    total,
    SUM(total) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL 1 DAY PRECEDING AND INTERVAL 1 DAY FOLLOWING
    ) AS rolling_3day
FROM orders
ORDER BY order_date;
```

This sums all orders within a 3-day window centered on each row's date, grouping by actual date value rather than row position.

## When to Use RANGE vs ROWS

```text
Use RANGE when:
- Tied ORDER BY values should be treated as a single logical group
- You want date/value-based windows (e.g., "all rows within 7 days")
- The default implicit frame behavior is desired

Use ROWS when:
- You need each physical row to accumulate independently
- You want exact N-row windows regardless of value ties
- Building running totals where each row has a distinct value
```

## Practical Example - Daily Revenue with Date-Based Window

Sum all revenue for orders within 2 days before through the current date:

```sql
SELECT
    order_date,
    total,
    SUM(total) OVER (
        ORDER BY order_date
        RANGE BETWEEN INTERVAL 2 DAY PRECEDING AND CURRENT ROW
    ) AS rolling_sum_2day
FROM orders
ORDER BY order_date;
```

## Summary

`RANGE BETWEEN` in MySQL window functions uses logical value-based boundaries, treating tied `ORDER BY` values as equivalent peers within the same frame boundary. This differs from `ROWS BETWEEN`, which counts physical rows individually. Use `RANGE` when you want date or numeric range-based windows, or when tied values should produce identical aggregate results. `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is also the implicit default frame when `ORDER BY` is specified without an explicit frame clause.
