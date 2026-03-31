# How to Use ROW_NUMBER() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, ROW_NUMBER, Ranking, SQL

Description: Learn how to use ROW_NUMBER() window function in ClickHouse to assign sequential row numbers within partitions for deduplication, ranking, and pagination.

---

## What Is ROW_NUMBER()

`ROW_NUMBER()` is a window function that assigns a unique sequential integer to each row within a partition, ordered by a specified column. Row numbers start at 1 and increment by 1 for each row.

```sql
ROW_NUMBER() OVER (
    [PARTITION BY partition_column]
    ORDER BY sort_column [ASC|DESC]
)
```

## Basic ROW_NUMBER() Example

```sql
CREATE TABLE sales (
    sale_id UInt64,
    salesperson String,
    region LowCardinality(String),
    amount Float64,
    sale_date Date
) ENGINE = MergeTree()
ORDER BY (sale_date, sale_id);

-- Assign row numbers ordered by sale amount descending
SELECT
    sale_id,
    salesperson,
    amount,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS rank_by_amount
FROM sales;
```

## ROW_NUMBER() with PARTITION BY

Use `PARTITION BY` to restart numbering for each group:

```sql
-- Rank sales within each region
SELECT
    salesperson,
    region,
    amount,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) AS regional_rank
FROM sales;

-- Result: each region has its own 1, 2, 3, ... numbering
```

## Deduplication with ROW_NUMBER()

One of the most common uses: keep only the latest record per key:

```sql
CREATE TABLE user_events (
    user_id UInt64,
    event_type LowCardinality(String),
    ts DateTime,
    data String
) ENGINE = MergeTree()
ORDER BY (user_id, ts);

-- Keep only the most recent event per user per event type
SELECT user_id, event_type, ts, data
FROM (
    SELECT
        user_id,
        event_type,
        ts,
        data,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_type
            ORDER BY ts DESC
        ) AS rn
    FROM user_events
)
WHERE rn = 1;
```

## Getting Top-N per Group

```sql
-- Top 3 products by revenue per category
SELECT product_id, category, revenue
FROM (
    SELECT
        product_id,
        category,
        revenue,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM product_sales
)
WHERE rn <= 3
ORDER BY category, rn;
```

## ROW_NUMBER() for Pagination

```sql
-- Page 3, 20 items per page
SELECT *
FROM (
    SELECT
        order_id,
        customer_id,
        amount,
        order_date,
        ROW_NUMBER() OVER (ORDER BY order_date DESC, order_id) AS rn
    FROM orders
)
WHERE rn BETWEEN 41 AND 60  -- page 3: rows 41-60
ORDER BY rn;
```

## ROW_NUMBER() vs RANK() vs DENSE_RANK()

These three functions differ when rows have equal values:

```sql
SELECT
    player,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_num,   -- unique, no ties
    rank() OVER (ORDER BY score DESC) AS rnk,              -- ties share rank, gaps after
    dense_rank() OVER (ORDER BY score DESC) AS dense_rnk   -- ties share rank, no gaps
FROM leaderboard;

-- If scores are: 100, 100, 80, 75
-- ROW_NUMBER: 1, 2, 3, 4  (arbitrary tiebreak)
-- RANK:       1, 1, 3, 4  (gap after tie)
-- DENSE_RANK: 1, 1, 2, 3  (no gap)
```

## Practical Example: Finding First Purchase per User

```sql
CREATE TABLE purchases (
    purchase_id UInt64,
    user_id UInt64,
    product_id UInt32,
    amount Float64,
    purchased_at DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, purchased_at);

-- First purchase for each user
SELECT
    user_id,
    product_id,
    amount,
    purchased_at AS first_purchase_date
FROM (
    SELECT
        user_id,
        product_id,
        amount,
        purchased_at,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY purchased_at ASC) AS rn
    FROM purchases
)
WHERE rn = 1;

-- Analyze first-purchase patterns
SELECT
    product_id,
    count() AS times_bought_first,
    round(avg(amount), 2) AS avg_first_purchase_value
FROM (
    SELECT
        user_id,
        product_id,
        amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY purchased_at ASC) AS rn
    FROM purchases
)
WHERE rn = 1
GROUP BY product_id
ORDER BY times_bought_first DESC
LIMIT 10;
```

## Performance Considerations

```sql
-- ROW_NUMBER() requires sorting within partitions
-- Ensure ORDER BY columns match or are a prefix of the table's ORDER BY for performance

-- Set memory limit for window functions
SET max_bytes_before_external_sort = 10000000000;  -- spill to disk if needed

-- Check if window functions are using the table's sort order
EXPLAIN SELECT ROW_NUMBER() OVER (ORDER BY sale_date) FROM sales;
```

## Summary

`ROW_NUMBER()` in ClickHouse assigns sequential integers to rows within window partitions, starting at 1. It is essential for deduplication (keep only the latest row per key), top-N-per-group queries, and pagination. Unlike `RANK()` and `DENSE_RANK()`, `ROW_NUMBER()` always assigns unique numbers with no ties. Use `PARTITION BY` to restart numbering within each group and `ORDER BY` to control the ordering within each partition.
