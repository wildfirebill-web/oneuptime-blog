# How to Calculate Percentile Rank in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Percentile Rank, PERCENT_RANK, Window Function, Analytics

Description: Learn how to calculate percentile rank in ClickHouse using percent_rank, ntile, and quantile functions to position values within a distribution.

---

## What is Percentile Rank?

Percentile rank tells you what percentage of values in a dataset fall at or below a given value. A score at the 90th percentile means 90% of all scores are equal to or lower. ClickHouse provides several functions for this calculation.

## Sample Test Scores Table

```sql
CREATE TABLE student_scores
(
    student_id UInt32,
    subject String,
    score Float64
)
ENGINE = MergeTree()
ORDER BY (subject, student_id);

INSERT INTO student_scores VALUES
(1, 'Math', 85), (2, 'Math', 92), (3, 'Math', 78),
(4, 'Math', 95), (5, 'Math', 70), (6, 'Math', 88),
(7, 'Math', 61), (8, 'Math', 73), (9, 'Math', 90), (10, 'Math', 55);
```

## Method 1 - percent_rank Window Function

```sql
SELECT
    student_id,
    score,
    round(percent_rank() OVER (PARTITION BY subject ORDER BY score), 4) AS percentile_rank
FROM student_scores
ORDER BY subject, score;
```

`percent_rank` returns 0 for the lowest value and 1 for the highest.

## Method 2 - rank() Based Percentile

```sql
SELECT
    student_id,
    subject,
    score,
    rank() OVER (PARTITION BY subject ORDER BY score) AS rank_asc,
    round(
        (rank() OVER (PARTITION BY subject ORDER BY score) - 1) * 100.0 /
        (count() OVER (PARTITION BY subject) - 1),
        1
    ) AS percentile_rank_pct
FROM student_scores
ORDER BY subject, score;
```

## Method 3 - Self-Join Count

Classic approach without window functions:

```sql
SELECT
    s1.student_id,
    s1.subject,
    s1.score,
    round(
        countIf(s2.score <= s1.score) * 100.0 / count(),
        1
    ) AS percentile_rank_pct
FROM student_scores s1
JOIN student_scores s2 ON s1.subject = s2.subject
GROUP BY s1.student_id, s1.subject, s1.score
ORDER BY s1.subject, s1.score;
```

## NTILE for Bucket Assignment

Assign each row to a bucket (quartile, decile, percentile):

```sql
-- Quartiles (4 buckets)
SELECT
    student_id,
    score,
    ntile(4) OVER (ORDER BY score) AS quartile
FROM student_scores
ORDER BY score;

-- Deciles (10 buckets)
SELECT
    student_id,
    score,
    ntile(10) OVER (ORDER BY score) AS decile
FROM student_scores
ORDER BY score;
```

## Finding the Score at a Given Percentile

```sql
SELECT
    subject,
    quantile(0.25)(score) AS p25,
    quantile(0.50)(score) AS p50_median,
    quantile(0.75)(score) AS p75,
    quantile(0.90)(score) AS p90
FROM student_scores
GROUP BY subject;
```

## Ranking Users by Revenue Percentile

```sql
SELECT
    user_id,
    total_spend,
    round(percent_rank() OVER (ORDER BY total_spend) * 100, 1) AS spend_percentile,
    ntile(100) OVER (ORDER BY total_spend) AS spend_centile
FROM (
    SELECT user_id, sum(amount) AS total_spend
    FROM orders
    GROUP BY user_id
)
ORDER BY total_spend DESC
LIMIT 20;
```

## Summary

ClickHouse calculates percentile rank via the `percent_rank()` window function, `rank()` arithmetic, or `ntile()` for bucket assignment. Use `quantile(p)(column)` to find the value at a given percentile, and `percent_rank()` to find where a given value sits within the distribution.
