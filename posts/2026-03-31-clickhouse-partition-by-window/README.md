# How to Use PARTITION BY with Window Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Analytics, Partition

Description: PARTITION BY divides result rows into independent groups before applying window functions, resetting calculations per group. Learn with running totals and rankings.

---

Without PARTITION BY, a window function treats the entire result set as one group. PARTITION BY lets you carve that result set into independent sub-groups, called partitions, and apply the window function separately inside each one. Rank resets to 1 for every new partition, running totals restart from zero, and row numbers count from 1 again. This is the primary mechanism for per-entity analytics in ClickHouse.

## Basic Syntax

```text
<window_function>() OVER (
    PARTITION BY <column1> [, <column2>, ...]
    [ORDER BY <col> [ASC|DESC]]
    [ROWS|RANGE BETWEEN ...]
)
```

The PARTITION BY clause comes first inside OVER(), followed by ORDER BY and any frame specification.

## Setting Up Sample Data

Create tables to demonstrate common PARTITION BY use cases.

```sql
CREATE TABLE user_purchases
(
    user_id    UInt32,
    purchase_date Date,
    amount     Float64,
    category   String
)
ENGINE = MergeTree()
ORDER BY (user_id, purchase_date);

INSERT INTO user_purchases VALUES
    (1, '2024-01-01', 50,  'Electronics'),
    (1, '2024-01-05', 120, 'Electronics'),
    (1, '2024-01-10', 30,  'Books'),
    (2, '2024-01-02', 80,  'Books'),
    (2, '2024-01-06', 200, 'Electronics'),
    (2, '2024-01-09', 60,  'Books'),
    (3, '2024-01-03', 40,  'Books'),
    (3, '2024-01-07', 90,  'Electronics'),
    (3, '2024-01-11', 110, 'Electronics');
```

## Per-User Running Totals

Without PARTITION BY, a running total would span all users. Adding `PARTITION BY user_id` resets the cumulative sum for each user.

```sql
SELECT
    user_id,
    purchase_date,
    amount,
    sum(amount) OVER (
        PARTITION BY user_id
        ORDER BY purchase_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS user_running_total
FROM user_purchases
ORDER BY user_id, purchase_date;
```

User 1's running total resets when user 2's rows begin. Each user independently accumulates their own spend over time.

## Per-Category Rankings

rank() assigns a rank within each partition. Here each category gets its own independent ranking by purchase amount.

```sql
SELECT
    category,
    user_id,
    amount,
    rank() OVER (
        PARTITION BY category
        ORDER BY amount DESC
    ) AS category_rank
FROM user_purchases
ORDER BY category, category_rank;
```

The highest-amount purchase within Electronics is ranked 1, and the highest within Books is also ranked 1. The two partitions are completely independent.

## Row Numbers per User

row_number() restarts at 1 for each new partition. This is useful for labelling the Nth purchase per user.

```sql
SELECT
    user_id,
    purchase_date,
    amount,
    row_number() OVER (
        PARTITION BY user_id
        ORDER BY purchase_date
    ) AS purchase_sequence
FROM user_purchases
ORDER BY user_id, purchase_date;
```

This tells you whether a purchase was the user's first, second, third, and so on - useful for funnel analysis and cohort reporting.

## Identifying the Latest Purchase per User

Combining row_number() with a subquery lets you filter to only the most recent row per partition.

```sql
SELECT user_id, purchase_date, amount, category
FROM (
    SELECT
        user_id,
        purchase_date,
        amount,
        category,
        row_number() OVER (
            PARTITION BY user_id
            ORDER BY purchase_date DESC
        ) AS rn
    FROM user_purchases
)
WHERE rn = 1
ORDER BY user_id;
```

Only the row with `rn = 1` (the most recent purchase) is returned for each user, making this a clean "latest record per group" pattern.

## Multiple PARTITION BY Columns

You can partition on a combination of columns. This example computes the running total per user per category.

```sql
SELECT
    user_id,
    category,
    purchase_date,
    amount,
    sum(amount) OVER (
        PARTITION BY user_id, category
        ORDER BY purchase_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS user_category_running_total
FROM user_purchases
ORDER BY user_id, category, purchase_date;
```

User 1's Electronics running total and User 1's Books running total are tracked independently. Adding the second column tightens the partition further.

## Per-Partition Average for Comparison

You can compute the partition-level aggregate alongside each row to compare a row's value to its group average.

```sql
SELECT
    user_id,
    purchase_date,
    amount,
    round(avg(amount) OVER (PARTITION BY user_id), 2) AS user_avg_spend,
    round(amount - avg(amount) OVER (PARTITION BY user_id), 2) AS deviation_from_avg
FROM user_purchases
ORDER BY user_id, purchase_date;
```

When no ORDER BY or frame is given inside OVER(), the window covers the entire partition. The result is each user's average spend repeated on every one of their rows, making it straightforward to see which purchases were above or below average.

## Combining PARTITION BY with Dense Rank

dense_rank() eliminates gaps in the ranking sequence when ties occur. This example ranks users within each category by their total spend.

```sql
SELECT
    category,
    user_id,
    total_spend,
    dense_rank() OVER (
        PARTITION BY category
        ORDER BY total_spend DESC
    ) AS spend_rank
FROM (
    SELECT
        user_id,
        category,
        sum(amount) AS total_spend
    FROM user_purchases
    GROUP BY user_id, category
)
ORDER BY category, spend_rank;
```

If two users have the same total spend in a category, they share a rank and the next rank is not skipped.

## Summary

PARTITION BY is the mechanism that makes window functions truly useful for per-entity analytics. It splits the result set into independent groups, ensuring that rankings, running totals, and row numbers reset cleanly for each new group. You can partition on a single column for simple groupings or on multiple columns for finer-grained sub-partitions. The pattern of computing row_number() and then filtering on rn = 1 is especially powerful for retrieving the latest or top record per group without a self-join.
