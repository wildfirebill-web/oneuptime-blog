# How to Use ROW_NUMBER() Window Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Analytics, Query Optimization

Description: Learn how ROW_NUMBER() assigns sequential integers to rows within a partition, with practical examples for ranking items, deduplicating records, and pagination.

---

ClickHouse supports window functions as of version 21.1, and `ROW_NUMBER()` is one of the most commonly used. It assigns a unique sequential integer to each row within a partition, restarting from 1 for each new partition group. Unlike `RANK()` or `DENSE_RANK()`, `ROW_NUMBER()` never produces ties - every row gets a distinct number, making it especially useful for deduplication and selecting exactly the top-N rows per group.

## Syntax

The general form of `ROW_NUMBER()` uses the `OVER` clause to define the window:

```text
ROW_NUMBER() OVER (
    [PARTITION BY partition_expr [, ...]]
    ORDER BY sort_expr [ASC | DESC] [, ...]
)
```

`PARTITION BY` divides the result set into independent groups. `ORDER BY` determines the sequence within each partition. When `PARTITION BY` is omitted, the entire result set is treated as one partition.

## Basic Example: Numbering All Rows

The simplest use case numbers every row in the result set ordered by a single column. This is useful for generating surrogate row numbers for display or pagination.

```sql
SELECT
    order_id,
    customer_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (ORDER BY order_date ASC) AS row_num
FROM orders
ORDER BY row_num;
```

Each row receives a number from 1 upward in the order of `order_date`. Two rows with the same `order_date` will still receive different numbers - the tie-breaking order is non-deterministic unless additional sort columns are specified.

## Ranking Items Per Partition

The real power of `ROW_NUMBER()` appears when combined with `PARTITION BY`. Here, each user's orders are numbered independently starting from 1:

```sql
SELECT
    customer_id,
    order_id,
    order_date,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY order_date ASC
    ) AS order_seq
FROM orders
ORDER BY customer_id, order_seq;
```

Customer A's orders are numbered 1, 2, 3 and customer B's orders are also numbered 1, 2, 3 - completely independently of each other.

## Selecting Top-N Per Group

A very common pattern is to keep only the most recent or highest-value N records per group. Wrap the window function in a subquery (or CTE) and filter on the row number.

The query below retrieves the 3 most recent orders for each customer:

```sql
SELECT
    customer_id,
    order_id,
    order_date,
    amount
FROM (
    SELECT
        customer_id,
        order_id,
        order_date,
        amount,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC
        ) AS rn
    FROM orders
)
WHERE rn <= 3
ORDER BY customer_id, order_date DESC;
```

ClickHouse evaluates the inner `SELECT` first and then applies the `WHERE rn <= 3` filter. This is more efficient than using `LIMIT` on the outer query when you need per-partition limits.

## Deduplicating Rows

When a table has duplicate records (same business key appearing multiple times), `ROW_NUMBER()` provides a clean way to keep exactly one copy per key. Suppose `events` has duplicate `(session_id, event_type)` combinations and you want only the earliest occurrence:

```sql
SELECT
    session_id,
    event_type,
    event_time,
    properties
FROM (
    SELECT
        session_id,
        event_type,
        event_time,
        properties,
        ROW_NUMBER() OVER (
            PARTITION BY session_id, event_type
            ORDER BY event_time ASC
        ) AS rn
    FROM events
)
WHERE rn = 1
ORDER BY session_id, event_time;
```

By partitioning on the business key and ordering by `event_time ASC`, the earliest row gets `rn = 1` and the filter removes all duplicates. Changing `ASC` to `DESC` keeps the latest occurrence instead.

## Pagination with ROW_NUMBER()

For offset-based pagination across a large result set, `ROW_NUMBER()` gives you stable row positions:

```sql
-- Return rows 21 through 40 (page 2, page size 20)
SELECT
    product_id,
    product_name,
    price,
    row_num
FROM (
    SELECT
        product_id,
        product_name,
        price,
        ROW_NUMBER() OVER (ORDER BY price DESC, product_id ASC) AS row_num
    FROM products
    WHERE category = 'Electronics'
)
WHERE row_num BETWEEN 21 AND 40
ORDER BY row_num;
```

Adding `product_id ASC` as a secondary sort key ensures the ordering is deterministic even when multiple products share the same price.

## Comparing ROW_NUMBER() with RANK() and DENSE_RANK()

To understand when `ROW_NUMBER()` is the right choice versus alternatives, consider this side-by-side comparison using a scores table:

```sql
SELECT
    player_name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_number,
    RANK()       OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank
FROM player_scores
ORDER BY score DESC;
```

Given scores 100, 100, 90, 80:

```text
player_name | score | row_number | rank | dense_rank
------------|-------|------------|------|------------
Alice       |   100 |          1 |    1 |          1
Bob         |   100 |          2 |    1 |          1
Carol       |    90 |          3 |    3 |          2
Dave        |    80 |          4 |    4 |          3
```

`ROW_NUMBER()` always produces unique numbers. `RANK()` skips position 2 after a tie. `DENSE_RANK()` never skips positions. Choose `ROW_NUMBER()` when you need exactly one row per partition position.

## Performance Considerations

ClickHouse window functions require a sort step internally. For large tables, ensure the data is stored (or pre-sorted) in an order compatible with your `PARTITION BY` and `ORDER BY` columns.

When the source table uses a `MergeTree` engine with a matching `ORDER BY` key, ClickHouse can leverage pre-sorted data:

```sql
-- If the table is defined with ORDER BY (customer_id, order_date),
-- this window function benefits from pre-sorted storage
SELECT
    customer_id,
    order_id,
    amount,
    ROW_NUMBER() OVER (
        PARTITION BY customer_id
        ORDER BY order_date DESC
    ) AS rn
FROM orders_mergetree
WHERE order_date >= today() - 30;
```

For ad-hoc analytics on large unordered datasets, consider materializing intermediate results or using `FINAL` on `ReplacingMergeTree` tables as an alternative to window-based deduplication.

## Summary

`ROW_NUMBER()` is a fundamental window function in ClickHouse that assigns a unique sequential integer to each row within a partition. Its main strengths are deduplication (keep only `rn = 1`), top-N-per-group selection (filter `rn <= N`), and stable pagination. Because it never produces ties, it is the right choice whenever you need exactly one row per rank position. Combine it with `PARTITION BY` to apply independent numbering across groups, and always include a deterministic `ORDER BY` to ensure reproducible results.
