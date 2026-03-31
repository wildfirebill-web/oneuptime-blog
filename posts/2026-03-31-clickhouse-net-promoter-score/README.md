# How to Calculate Net Promoter Score (NPS) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NPS, Analytics, Metric, SQL

Description: Calculate Net Promoter Score from survey response data in ClickHouse using SQL aggregations to segment promoters, passives, and detractors.

---

## NPS Formula

NPS = (% Promoters) - (% Detractors)

- Score 9-10: Promoters
- Score 7-8: Passives
- Score 0-6: Detractors

NPS ranges from -100 to +100.

## Table Schema

```sql
CREATE TABLE nps_responses (
    response_id UUID,
    user_id     UInt64,
    score       UInt8,        -- 0 to 10
    submitted_at DateTime
) ENGINE = MergeTree()
ORDER BY (submitted_at, user_id);
```

## Basic NPS Calculation

```sql
SELECT
    countIf(score >= 9)                         AS promoters,
    countIf(score >= 7 AND score <= 8)          AS passives,
    countIf(score <= 6)                         AS detractors,
    count()                                     AS total,
    round(
        (countIf(score >= 9) - countIf(score <= 6))
        * 100.0 / count(),
        1
    )                                           AS nps
FROM nps_responses
WHERE submitted_at >= today() - 90;
```

## NPS Over Time (Monthly Trend)

```sql
SELECT
    toStartOfMonth(submitted_at)                AS month,
    round(
        (countIf(score >= 9) - countIf(score <= 6))
        * 100.0 / count(),
        1
    )                                           AS nps,
    count()                                     AS responses
FROM nps_responses
WHERE submitted_at >= today() - 365
GROUP BY month
ORDER BY month;
```

## NPS by Product Segment

```sql
SELECT
    segment,
    round(
        (countIf(score >= 9) - countIf(score <= 6))
        * 100.0 / count(),
        1
    ) AS nps,
    count() AS n
FROM nps_responses
JOIN users USING (user_id)
WHERE submitted_at >= today() - 90
GROUP BY segment
ORDER BY nps DESC;
```

## Confidence Interval for NPS

For statistical significance, include the sample size and compute a margin of error in your application layer. Rule of thumb: NPS comparisons need at least 200 responses per segment.

## Score Distribution

```sql
SELECT
    score,
    count()                        AS cnt,
    round(count() * 100.0 / sum(count()) OVER (), 1) AS pct
FROM nps_responses
WHERE submitted_at >= today() - 90
GROUP BY score
ORDER BY score;
```

## Deduplicating Repeat Responses

If users can submit multiple times, keep only the most recent response per user.

```sql
SELECT
    argMax(score, submitted_at) AS latest_score
FROM nps_responses
WHERE submitted_at >= today() - 90
GROUP BY user_id
```

Wrap this in a subquery or CTE before calculating NPS.

## Summary

ClickHouse calculates NPS efficiently with `countIf` for segment counting. Run monthly trend queries to track movement over time, add `GROUP BY` for segment breakdowns, and deduplicate repeat responses with `argMax` before computing the final score.
