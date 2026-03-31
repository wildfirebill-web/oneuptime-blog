# How to Use AVG() OVER for Moving Averages in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, Analytics, SQL

Description: Learn how to calculate moving averages in MySQL using AVG() with OVER and window frame specifications to smooth time-series data.

---

## What Is a Moving Average?

A moving average computes the average of a value over a sliding window of rows, smoothing out short-term fluctuations in time-series data. MySQL 8.0+ supports this natively via `AVG()` as a window function.

## Basic Syntax

```sql
AVG(expression) OVER (
    [PARTITION BY column, ...]
    ORDER BY column
    ROWS BETWEEN N PRECEDING AND CURRENT ROW
)
```

`N PRECEDING AND CURRENT ROW` defines a window of `N+1` rows ending at the current row.

## Sample Table

```sql
CREATE TABLE stock_prices (
    price_date DATE,
    ticker     VARCHAR(10),
    close_price DECIMAL(10,2)
);

INSERT INTO stock_prices VALUES
  ('2024-01-01', 'ACME', 100.00),
  ('2024-01-02', 'ACME', 102.50),
  ('2024-01-03', 'ACME',  98.00),
  ('2024-01-04', 'ACME', 105.00),
  ('2024-01-05', 'ACME', 110.00),
  ('2024-01-06', 'ACME', 107.00),
  ('2024-01-07', 'ACME', 112.00);
```

## 3-Day Moving Average

```sql
SELECT
    price_date,
    ticker,
    close_price,
    ROUND(
        AVG(close_price) OVER (
            PARTITION BY ticker
            ORDER BY price_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2
    ) AS moving_avg_3d
FROM stock_prices
ORDER BY price_date;
```

Output:

```text
price_date  | ticker | close_price | moving_avg_3d
------------+--------+-------------+--------------
2024-01-01  | ACME   |      100.00 |        100.00
2024-01-02  | ACME   |      102.50 |        101.25
2024-01-03  | ACME   |       98.00 |        100.17
2024-01-04  | ACME   |      105.00 |        101.83
2024-01-05  | ACME   |      110.00 |        104.33
2024-01-06  | ACME   |      107.00 |        107.33
2024-01-07  | ACME   |      112.00 |        109.67
```

The first rows have fewer than 3 preceding rows, so MySQL uses however many rows are available.

## 7-Day Moving Average

```sql
SELECT
    price_date,
    ticker,
    close_price,
    ROUND(
        AVG(close_price) OVER (
            PARTITION BY ticker
            ORDER BY price_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ), 2
    ) AS moving_avg_7d
FROM stock_prices
ORDER BY price_date;
```

## Centered Moving Average

A centered moving average uses rows both before and after the current row:

```sql
SELECT
    price_date,
    close_price,
    ROUND(
        AVG(close_price) OVER (
            ORDER BY price_date
            ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
        ), 2
    ) AS centered_avg_3d
FROM stock_prices;
```

This gives a symmetric average of the previous row, current row, and next row.

## Moving Average with Minimum Row Count

MySQL does not have a built-in minimum row threshold, but you can filter with a subquery:

```sql
SELECT price_date, ticker, close_price, moving_avg_3d
FROM (
    SELECT
        price_date,
        ticker,
        close_price,
        ROW_NUMBER() OVER (PARTITION BY ticker ORDER BY price_date) AS rn,
        AVG(close_price) OVER (
            PARTITION BY ticker
            ORDER BY price_date
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS moving_avg_3d
    FROM stock_prices
) t
WHERE rn >= 3;
```

This returns only rows where at least 3 data points were available.

## Comparing Multiple Moving Windows

```sql
SELECT
    price_date,
    close_price,
    ROUND(AVG(close_price) OVER (ORDER BY price_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS ma_3d,
    ROUND(AVG(close_price) OVER (ORDER BY price_date ROWS BETWEEN 4 PRECEDING AND CURRENT ROW), 2) AS ma_5d,
    ROUND(AVG(close_price) OVER (ORDER BY price_date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS ma_7d
FROM stock_prices
ORDER BY price_date;
```

Displaying multiple moving averages alongside each other is a classic pattern for identifying trend crossovers.

## Summary

`AVG() OVER` with a `ROWS BETWEEN N PRECEDING AND CURRENT ROW` frame computes a trailing moving average in MySQL 8.0+. Use `PARTITION BY` to compute independent moving averages per group, and adjust the window size to control smoothing. For centered averages, extend the frame to include following rows. Filter by `ROW_NUMBER()` when you need a minimum window size before averaging.
