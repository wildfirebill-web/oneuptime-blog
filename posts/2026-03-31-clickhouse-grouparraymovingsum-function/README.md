# How to Use groupArrayMovingSum() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, GroupArrayMovingSum, Window

Description: Learn how to use groupArrayMovingSum() in ClickHouse to compute cumulative and rolling sums over ordered arrays, with window size and practical examples.

---

`groupArrayMovingSum(column)` is a ClickHouse aggregate function that first collects column values into an array (like `groupArray`), then computes a running (cumulative) sum over that array. The optional window size variant `groupArrayMovingSum(window)(column)` limits the sum to the most recent `window` elements, turning it into a sliding window sum. This is one of the most efficient ways to compute moving aggregations in ClickHouse without window functions.

## Basic groupArrayMovingSum()

Without a window size, `groupArrayMovingSum(col)` computes a cumulative sum - each element in the output array is the sum of all previous elements plus the current one.

```sql
CREATE TABLE daily_sales
(
    product   String,
    sale_date Date,
    revenue   Float64
)
ENGINE = MergeTree()
ORDER BY (product, sale_date);

INSERT INTO daily_sales VALUES
    ('widget', '2026-03-25', 100.0),
    ('widget', '2026-03-26', 150.0),
    ('widget', '2026-03-27',  80.0),
    ('widget', '2026-03-28', 200.0),
    ('widget', '2026-03-29', 120.0),
    ('gadget', '2026-03-25', 500.0),
    ('gadget', '2026-03-26', 300.0),
    ('gadget', '2026-03-27', 450.0);

-- Cumulative revenue per product (ordered by date)
SELECT
    product,
    groupArray(sale_date)                        AS dates,
    groupArray(revenue)                          AS daily_revenue,
    groupArrayMovingSum(revenue)                 AS cumulative_revenue
FROM (
    SELECT product, sale_date, revenue
    FROM daily_sales
    ORDER BY product, sale_date
)
GROUP BY product;
```

Output for `widget`:
```text
daily_revenue:      [100, 150, 80, 200, 120]
cumulative_revenue: [100, 250, 330, 530, 650]
```

The input ordering matters - `groupArrayMovingSum` processes values in the order they appear in the array collected by `GROUP BY`. Always pre-sort with a subquery or `ORDER BY` in the inner query to ensure deterministic results.

## groupArrayMovingSum(window)(column) - Sliding Window

`groupArrayMovingSum(window)(column)` sums only the last `window` elements at each position, creating a sliding window.

```sql
-- 3-day rolling sum of revenue per product
SELECT
    product,
    groupArray(sale_date)                        AS dates,
    groupArray(revenue)                          AS daily_revenue,
    groupArrayMovingSum(3)(revenue)              AS rolling_3d_sum
FROM (
    SELECT product, sale_date, revenue
    FROM daily_sales
    ORDER BY product, sale_date
)
GROUP BY product;
```

Output for `widget`:
```text
daily_revenue:   [100, 150, 80, 200, 120]
rolling_3d_sum:  [100, 250, 330, 430, 400]
```

- Position 0: 100 (only 1 element available)
- Position 1: 100 + 150 = 250 (2 elements)
- Position 2: 100 + 150 + 80 = 330 (full 3-element window)
- Position 3: 150 + 80 + 200 = 430 (window slides: drops 100, adds 200)
- Position 4: 80 + 200 + 120 = 400 (window slides: drops 150, adds 120)

## Combining with groupArray for Date-Value Pairs

To return both dates and values in the same query result, collect them together using `groupArray`:

```sql
SELECT
    product,
    arrayZip(
        groupArray(sale_date),
        groupArrayMovingSum(3)(revenue)
    ) AS date_rolling_sum_pairs
FROM (
    SELECT product, sale_date, revenue
    FROM daily_sales
    ORDER BY product, sale_date
)
GROUP BY product;
```

## Expanding Results with arrayJoin

Use `arrayJoin` to expand the moving sum array back into rows for further processing or joining with other tables:

```sql
SELECT
    product,
    date,
    daily_rev,
    moving_sum
FROM (
    SELECT
        product,
        groupArray(sale_date)            AS dates,
        groupArray(revenue)              AS daily_revs,
        groupArrayMovingSum(3)(revenue)  AS moving_sums
    FROM (
        SELECT product, sale_date, revenue
        FROM daily_sales
        ORDER BY product, sale_date
    )
    GROUP BY product
)
ARRAY JOIN
    dates       AS date,
    daily_revs  AS daily_rev,
    moving_sums AS moving_sum
ORDER BY product, date;
```

## groupArrayMovingAvg() - Moving Average Companion

`groupArrayMovingAvg(window)(column)` computes a sliding window average instead of a sum. It follows the same semantics:

```sql
-- 3-day moving average of revenue
SELECT
    product,
    groupArray(sale_date)                        AS dates,
    groupArrayMovingAvg(3)(revenue)              AS rolling_3d_avg
FROM (
    SELECT product, sale_date, revenue
    FROM daily_sales
    ORDER BY product, sale_date
)
GROUP BY product;
```

## Performance vs Window Functions

ClickHouse window functions (`SUM() OVER (...)`) are available but compute row-by-row. `groupArrayMovingSum` operates on arrays in a single aggregation pass, which can be faster for analytical batch queries where you need the full time series per group.

```sql
-- Window function approach (row-by-row, flexible)
SELECT
    product,
    sale_date,
    revenue,
    sum(revenue) OVER (
        PARTITION BY product
        ORDER BY sale_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS rolling_3d_sum
FROM daily_sales
ORDER BY product, sale_date;

-- groupArrayMovingSum approach (array-based, returns full series per group)
SELECT
    product,
    groupArrayMovingSum(3)(revenue) AS rolling_3d_sum_array
FROM (SELECT product, sale_date, revenue FROM daily_sales ORDER BY product, sale_date)
GROUP BY product;
```

Use window functions when you need per-row output with other non-aggregated columns. Use `groupArrayMovingSum` when you want the full series as an array for downstream array processing.

## Summary

`groupArrayMovingSum(col)` computes a cumulative sum over column values in a group, producing an array where each element is the running total up to that position. `groupArrayMovingSum(N)(col)` limits the window to the most recent `N` elements, creating a sliding sum. Always sort data before aggregating to ensure the array ordering is deterministic. Pair with `groupArray` and `arrayZip` to include date labels, or use `arrayJoin` to expand results back into rows.
