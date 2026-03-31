# How to Use RANK() and DENSE_RANK() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Analytics, Ranking

Description: Learn how RANK() and DENSE_RANK() assign rankings to rows in ClickHouse, including how they handle ties and when to choose each over ROW_NUMBER().

---

Ranking queries are a staple of analytics: leaderboards, top sellers, highest-scoring events. ClickHouse provides two window functions for this - `RANK()` and `DENSE_RANK()` - that differ only in how they treat tied values. Understanding the difference is crucial for producing correct results in competitive ranking scenarios.

## Syntax

Both functions share the same `OVER` clause structure:

```text
RANK()       OVER ([PARTITION BY ...] ORDER BY ...)
DENSE_RANK() OVER ([PARTITION BY ...] ORDER BY ...)
```

Neither function takes any arguments; the ranking is derived entirely from the `ORDER BY` expression inside the `OVER` clause.

## The Key Difference: Gaps vs. No Gaps

`RANK()` leaves gaps after tied positions. If three rows tie for rank 1, the next rank assigned is 4 (1 + 3 tied rows). `DENSE_RANK()` never leaves gaps - after three rows tied at rank 1, the next rank is 2.

The following query demonstrates both side by side:

```sql
SELECT
    player_name,
    score,
    RANK()       OVER (ORDER BY score DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank,
    ROW_NUMBER() OVER (ORDER BY score DESC) AS row_number
FROM player_scores
ORDER BY score DESC;
```

Given the scores 500, 500, 480, 460, 460, 420:

```text
player_name | score | rank | dense_rank | row_number
------------|-------|------|------------|------------
Alice       |   500 |    1 |          1 |          1
Bob         |   500 |    1 |          1 |          2
Carol       |   480 |    3 |          2 |          3
Dave        |   460 |    4 |          3 |          4
Eve         |   460 |    4 |          3 |          5
Frank       |   420 |    6 |          4 |          6
```

With `RANK()`, positions 2 and 5 are skipped. With `DENSE_RANK()`, every integer from 1 to 4 is used. `ROW_NUMBER()` always produces unique sequential numbers regardless of ties.

## Leaderboard Per Category

Combine `PARTITION BY` with `RANK()` to build per-category leaderboards. This query ranks products by revenue within each product category:

```sql
SELECT
    category,
    product_name,
    revenue,
    RANK() OVER (
        PARTITION BY category
        ORDER BY revenue DESC
    ) AS revenue_rank
FROM product_sales
ORDER BY category, revenue_rank;
```

Each category restarts at rank 1. Two products with the same revenue in the same category will share the same rank, and the next rank will be skipped accordingly.

## Top-N Per Group Using RANK()

To retrieve the top 3 products per category, filter on the rank in a subquery:

```sql
SELECT
    category,
    product_name,
    revenue,
    revenue_rank
FROM (
    SELECT
        category,
        product_name,
        revenue,
        RANK() OVER (
            PARTITION BY category
            ORDER BY revenue DESC
        ) AS revenue_rank
    FROM product_sales
)
WHERE revenue_rank <= 3
ORDER BY category, revenue_rank;
```

Note: because `RANK()` can produce ties, this query may return more than 3 rows per category when multiple products share rank 3. If you want exactly 3 rows per category, use `ROW_NUMBER()` instead.

## Using DENSE_RANK() for Tier Classification

`DENSE_RANK()` is ideal when the rank number itself carries meaning, such as tier assignment. Here, customers are bucketed into tiers based on their total spend:

```sql
SELECT
    customer_id,
    customer_name,
    total_spend,
    DENSE_RANK() OVER (ORDER BY total_spend DESC) AS spend_rank,
    CASE
        WHEN DENSE_RANK() OVER (ORDER BY total_spend DESC) = 1 THEN 'Platinum'
        WHEN DENSE_RANK() OVER (ORDER BY total_spend DESC) <= 5 THEN 'Gold'
        WHEN DENSE_RANK() OVER (ORDER BY total_spend DESC) <= 20 THEN 'Silver'
        ELSE 'Bronze'
    END AS tier
FROM customer_summary
ORDER BY spend_rank;
```

Using `DENSE_RANK()` here ensures the tier thresholds (1, 5, 20) map to meaningful customer counts rather than arbitrary positional gaps.

## Ranking Across Multiple Partitions

A more complex real-world use case: rank sales representatives by closed deals within each region, then identify the top performer per region:

```sql
SELECT
    region,
    rep_name,
    closed_deals,
    deal_value,
    RANK() OVER (
        PARTITION BY region
        ORDER BY closed_deals DESC, deal_value DESC
    ) AS regional_rank
FROM sales_reps
ORDER BY region, regional_rank;
```

The secondary sort on `deal_value DESC` breaks ties in `closed_deals`, which makes the ranking more deterministic. Two reps with identical deals and deal value will still share a rank.

## Finding Rows with a Specific Rank

To find every row that occupies exactly rank 2 (the runners-up) within each partition:

```sql
SELECT
    category,
    product_name,
    revenue
FROM (
    SELECT
        category,
        product_name,
        revenue,
        DENSE_RANK() OVER (
            PARTITION BY category
            ORDER BY revenue DESC
        ) AS dr
    FROM product_sales
)
WHERE dr = 2
ORDER BY category, revenue DESC;
```

With `DENSE_RANK()`, rank 2 always exists as long as there are at least two distinct revenue values per category - there are no gaps to worry about.

## Choosing Between RANK(), DENSE_RANK(), and ROW_NUMBER()

```text
Scenario                                       Recommended Function
---------------------------------------------  --------------------
Need unique row position, no ties allowed      ROW_NUMBER()
Leaderboard where ties share a rank, gaps OK   RANK()
Tier/level assignment, no gaps desired         DENSE_RANK()
Exactly N rows per group                       ROW_NUMBER()
All tied rows at position N should be included RANK() or DENSE_RANK()
```

## Performance Notes

When running both `RANK()` and `DENSE_RANK()` in the same query, ClickHouse can share the sort step between them if they use the same `OVER` specification. Define them with identical window expressions to allow this optimization:

```sql
-- Both windows are identical - ClickHouse can reuse the sort
SELECT
    player_name,
    score,
    RANK()       OVER w AS r,
    DENSE_RANK() OVER w AS dr
FROM player_scores
WINDOW w AS (PARTITION BY game_id ORDER BY score DESC)
ORDER BY game_id, r;
```

The `WINDOW` clause (supported in ClickHouse) allows naming a window specification and reusing it, reducing both verbosity and computational overhead.

## Summary

`RANK()` and `DENSE_RANK()` are the go-to window functions for competitive ranking in ClickHouse. The core distinction is simple: `RANK()` leaves positional gaps after ties while `DENSE_RANK()` does not. Use `RANK()` when the position number must reflect "how many people scored above this row." Use `DENSE_RANK()` when the number should represent a tier or level without holes. Both support `PARTITION BY` for independent per-group rankings and work efficiently when the underlying table is sorted by the partition and order columns.
