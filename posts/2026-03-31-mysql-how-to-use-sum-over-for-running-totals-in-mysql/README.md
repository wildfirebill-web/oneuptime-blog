# How to Use SUM() OVER for Running Totals in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Functions, Analytics, Sql

Description: Learn how to compute running totals in MySQL using SUM() with an OVER clause and window frame specifications for cumulative calculations.

---

## What Is a Running Total?

A running total (also called a cumulative sum) adds each row's value to all previous rows' values to produce an ever-increasing total. In MySQL 8.0+, the `SUM()` window function combined with `OVER` and an appropriate window frame makes this straightforward.

## Basic Syntax

```sql
SUM(expression) OVER (
    [PARTITION BY column, ...]
    ORDER BY column [ASC|DESC]
    [ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW]
)
```

The default frame when `ORDER BY` is present is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which handles ties. Using `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is generally clearer for running totals.

## Sample Data

```sql
CREATE TABLE daily_revenue (
    sale_date DATE,
    region    VARCHAR(20),
    amount    DECIMAL(10,2)
);

INSERT INTO daily_revenue VALUES
  ('2024-01-01', 'North', 1200.00),
  ('2024-01-02', 'North',  800.00),
  ('2024-01-03', 'North', 1500.00),
  ('2024-01-01', 'South',  600.00),
  ('2024-01-02', 'South',  900.00),
  ('2024-01-03', 'South', 1100.00);
```

## Computing a Simple Running Total

```sql
SELECT
    sale_date,
    region,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_revenue
WHERE region = 'North'
ORDER BY sale_date;
```

Output:

```text
sale_date  | region | amount  | running_total
-----------+--------+---------+--------------
2024-01-01 | North  | 1200.00 |      1200.00
2024-01-02 | North  |  800.00 |      2000.00
2024-01-03 | North  | 1500.00 |      3500.00
```

## Running Total Per Partition

Use `PARTITION BY` to reset the running total for each region:

```sql
SELECT
    sale_date,
    region,
    amount,
    SUM(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM daily_revenue
ORDER BY region, sale_date;
```

Each region's running total starts fresh from its first row.

## Running Total with Percentage

Combine the running total with the overall total to show cumulative percentage:

```sql
SELECT
    sale_date,
    region,
    amount,
    SUM(amount) OVER (
        PARTITION BY region
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    ROUND(
        100.0 * SUM(amount) OVER (
            PARTITION BY region
            ORDER BY sale_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) / SUM(amount) OVER (PARTITION BY region),
        2
    ) AS pct_of_total
FROM daily_revenue
ORDER BY region, sale_date;
```

## Handling Ties in Dates

When multiple rows share the same `ORDER BY` value, use `ROWS` (physical rows) vs `RANGE` (logical range) carefully:

```sql
-- ROWS: each physical row gets a separate cumulative value
SUM(amount) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- RANGE: tied dates are all treated as part of the same window boundary
SUM(amount) OVER (ORDER BY sale_date RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

For running totals where ties should accumulate individually, use `ROWS`.

## Running Total Over All Rows (Grand Total in Every Row)

```sql
SELECT
    sale_date,
    region,
    amount,
    SUM(amount) OVER () AS grand_total
FROM daily_revenue;
```

Omitting `ORDER BY` and frame clause produces the full partition sum in every row.

## Summary

`SUM() OVER` with `ORDER BY` and `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is the standard pattern for running totals in MySQL 8.0+. Add `PARTITION BY` to compute independent running totals per group, and combine with the total sum to derive cumulative percentages. Choose `ROWS` over `RANGE` frames when you need per-row accumulation regardless of ties.
