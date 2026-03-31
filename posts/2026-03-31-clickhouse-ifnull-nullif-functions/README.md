# How to Use ifNull() and nullIf() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL Handling, Conditional Function, Data Cleaning

Description: Learn how to use ifNull() and nullIf() in ClickHouse for NULL replacement, sentinel value patterns, and safe division in analytics queries.

---

ClickHouse provides two complementary functions for working with NULL values. `ifNull(x, default)` returns `default` when `x` is NULL - this is equivalent to a two-argument `COALESCE`. `nullIf(x, y)` returns NULL when `x` equals `y`, otherwise returns `x`. Together they cover the most common NULL-handling patterns in data cleaning and transformation pipelines.

## ifNull() - Replace NULL with a Default

`ifNull(x, default)` is the simplest way to substitute a default value when a column is NULL.

```sql
-- Replace NULL phone numbers with a placeholder
SELECT
    user_id,
    ifNull(phone_number, 'not provided') AS phone
FROM users
LIMIT 10;

-- Replace NULL numeric values with 0
SELECT
    product_id,
    ifNull(discount_percent, 0) AS discount
FROM products
LIMIT 10;
```

## ifNull() for Multi-Source Fallback

When you have multiple potential sources for a value, use `ifNull` to try each in sequence.

```sql
-- Use billing address if shipping address is NULL
SELECT
    order_id,
    ifNull(shipping_address, billing_address) AS delivery_address
FROM orders
LIMIT 10;
```

## nullIf() - Create NULL from a Sentinel Value

`nullIf(x, y)` returns NULL when `x` equals `y`. This is useful when a column stores a sentinel value (like 0, -1, or 'N/A') to represent "no value" and you want to convert it to a proper NULL.

```sql
-- Convert sentinel -1 to NULL
SELECT
    user_id,
    nullIf(age, -1) AS clean_age
FROM users
LIMIT 10;

-- Convert empty string to NULL
SELECT
    user_id,
    nullIf(email, '') AS email_or_null
FROM users
LIMIT 10;
```

## Safe Division with nullIf

`nullIf(denominator, 0)` is the standard way to prevent division by zero. When the denominator is 0, `nullIf` returns NULL and the division result is NULL rather than throwing an error.

```sql
-- Safe division using nullIf
SELECT
    campaign_id,
    impressions,
    clicks,
    clicks / nullIf(impressions, 0) AS ctr
FROM campaign_stats
LIMIT 10;

-- Compute conversion rate safely
SELECT
    funnel_step,
    users_entered,
    users_converted,
    users_converted / nullIf(users_entered, 0) AS conversion_rate
FROM funnel_metrics
LIMIT 10;
```

## Combining ifNull and nullIf

You can chain them to convert sentinel values to NULL and then replace NULL with a different default.

```sql
-- Convert -1 sentinel to NULL, then replace NULL with 0
SELECT
    user_id,
    ifNull(nullIf(score, -1), 0) AS clean_score
FROM quiz_results
LIMIT 10;
```

## Data Quality Checks

Use `nullIf` to flag sentinel values as NULL so that aggregate functions naturally ignore them.

```sql
-- Average age, ignoring -1 sentinel values
SELECT
    avg(nullIf(age, -1)) AS avg_age,
    count(nullIf(age, -1)) AS users_with_age
FROM users;

-- Compare with and without sentinel filtering
SELECT
    avg(age)              AS avg_age_with_sentinel,
    avg(nullIf(age, -1))  AS avg_age_clean
FROM users;
```

## ifNull vs COALESCE vs if(isNull())

All three achieve the same result for two arguments. `ifNull` is the most concise.

```sql
-- These three are equivalent:
SELECT
    ifNull(email, 'no email')                        AS a,
    COALESCE(email, 'no email')                      AS b,
    if(isNull(email), 'no email', email)             AS c
FROM users
LIMIT 5;
```

For more than two alternatives, use `COALESCE`. For exactly two, `ifNull` is the clearest choice.

## nullIf for Conditional Aggregation

```sql
-- Count only non-zero revenue rows
SELECT
    product_id,
    count(nullIf(revenue, 0))     AS sales_with_revenue,
    avg(nullIf(revenue, 0))       AS avg_nonzero_revenue
FROM sales
GROUP BY product_id
ORDER BY sales_with_revenue DESC
LIMIT 10;
```

## Summary

`ifNull(x, default)` replaces NULL with a default value - use it whenever you need a fallback for missing data. `nullIf(x, y)` converts a specific sentinel value to NULL - use it to clean up columns that use 0, -1, or empty strings as stand-ins for "no value". The most important combined pattern is `clicks / nullIf(impressions, 0)` for safe division that returns NULL instead of throwing on a zero denominator.
