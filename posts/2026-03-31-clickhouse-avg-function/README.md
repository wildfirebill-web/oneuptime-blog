# How to Use avg() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Avg

Description: Learn how to use avg(), avgIf(), and avgWeighted() in ClickHouse, including NULL handling, precision considerations, and practical query examples.

---

`avg()` computes the arithmetic mean of a numeric column across all rows in a group. ClickHouse extends the standard function with `avgIf()` for conditional averages and `avgWeighted()` for weighted means. Understanding how each variant handles NULLs and floating-point precision helps you avoid subtle bugs in analytical queries.

## Basic avg()

`avg(column)` divides the sum of non-NULL values by the count of non-NULL rows.

```sql
CREATE TABLE product_reviews
(
    review_id   UInt64,
    product_id  UInt64,
    user_id     UInt64,
    rating      Nullable(Float32),
    review_date Date
)
ENGINE = MergeTree()
ORDER BY (review_date, review_id);

-- Average rating across all reviews
SELECT avg(rating) AS avg_rating FROM product_reviews;

-- Average rating per product
SELECT
    product_id,
    avg(rating)   AS avg_rating,
    count()       AS review_count
FROM product_reviews
GROUP BY product_id
ORDER BY avg_rating DESC;
```

The return type of `avg()` is always `Float64`, regardless of the input type.

## NULL Handling

`avg()` ignores NULL values - they are excluded from both the sum and the count used to compute the mean.

```sql
INSERT INTO product_reviews VALUES
    (1, 101, 1, 4.5, '2026-03-31'),
    (2, 101, 2, NULL, '2026-03-31'),  -- NULL rating - excluded
    (3, 101, 3, 3.0, '2026-03-31');

-- avg(rating) = (4.5 + 3.0) / 2 = 3.75, not (4.5 + 0 + 3.0) / 3
SELECT avg(rating) FROM product_reviews WHERE product_id = 101;
```

If you want NULLs to count as zero (treating missing ratings as the lowest score), use `ifNull()`:

```sql
-- Treat NULL as 0
SELECT avg(ifNull(rating, 0)) AS avg_with_nulls_as_zero
FROM product_reviews
WHERE product_id = 101;
```

## avgIf() - Conditional Average

`avgIf(column, condition)` computes the average only over rows where the condition is true. It is equivalent to a filtered subquery but executes in a single pass.

```sql
-- Average rating for reviews in 2026 vs overall
SELECT
    avg(rating)                               AS overall_avg,
    avgIf(rating, toYear(review_date) = 2026) AS avg_2026
FROM product_reviews;
```

Use `avgIf` to compute segment averages side by side:

```sql
-- Average rating by score bucket in one scan
SELECT
    product_id,
    avgIf(rating, rating >= 4)   AS avg_high_ratings,
    avgIf(rating, rating < 4)    AS avg_low_ratings,
    avg(rating)                   AS overall_avg
FROM product_reviews
GROUP BY product_id;
```

## avgWeighted() - Weighted Average

`avgWeighted(value, weight)` computes the weighted arithmetic mean: the sum of `value * weight` divided by the sum of `weight`. This is essential when each data point has a different significance.

```sql
CREATE TABLE course_grades
(
    student_id  UInt64,
    course      String,
    score       Float64,
    credits     UInt8   -- credit hours act as weights
)
ENGINE = MergeTree()
ORDER BY (student_id, course);

INSERT INTO course_grades VALUES
    (1, 'Math',    92.0, 4),
    (1, 'English', 85.0, 3),
    (1, 'PE',      95.0, 1);

-- Simple average: (92 + 85 + 95) / 3 = 90.67
SELECT avg(score)                    AS simple_avg FROM course_grades WHERE student_id = 1;

-- Weighted average: (92*4 + 85*3 + 95*1) / (4+3+1) = (368+255+95)/8 = 89.75
SELECT avgWeighted(score, credits)   AS weighted_gpa FROM course_grades WHERE student_id = 1;
```

`avgWeighted` is also useful for aggregating pre-aggregated data without losing fidelity:

```sql
-- Merge hourly averages into a daily average, weighted by sample count
SELECT
    toDate(ts) AS day,
    avgWeighted(hourly_avg, sample_count) AS daily_avg
FROM hourly_metrics
GROUP BY day;
```

## Precision Considerations

`avg()` always returns `Float64`. For financial or scientific data where precision matters, use `Decimal` types in combination with explicit sum and count:

```sql
-- High-precision average using Decimal arithmetic
SELECT
    toDecimal64(sum(toDecimal64(rating, 4)), 4) / count(rating) AS precise_avg
FROM product_reviews
WHERE product_id = 101;
```

Or use `sumKahan()` / manual computation if you are accumulating many floating-point values with significant rounding concerns:

```sql
-- Kahan-compensated sum for better float precision
SELECT sumKahan(rating) / count(rating) AS compensated_avg
FROM product_reviews;
```

## avg() in Time Series

A common pattern is computing rolling or periodic averages over time.

```sql
-- Weekly average rating trend per product
SELECT
    product_id,
    toMonday(review_date)   AS week,
    avg(rating)              AS weekly_avg_rating,
    count()                  AS review_count
FROM product_reviews
GROUP BY product_id, week
ORDER BY product_id, week;
```

```sql
-- Compare last 7 days avg vs last 30 days avg
SELECT
    product_id,
    avgIf(rating, review_date >= today() - 7)  AS avg_last_7d,
    avgIf(rating, review_date >= today() - 30) AS avg_last_30d
FROM product_reviews
GROUP BY product_id
ORDER BY product_id;
```

## Summary

`avg()` skips NULLs automatically - use `ifNull()` to treat them as a specific value instead. `avgIf()` lets you compute multiple conditional averages in a single scan without subqueries. Use `avgWeighted()` when data points carry different importance and a simple mean would be misleading. For high-precision use cases, consider explicit `Decimal` arithmetic instead of relying on `Float64`.
