# How to Build Top-N Tracking with Materialized Views in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Materialized View, Top-N Query, SummingMergeTree, Leaderboard

Description: Maintain real-time Top-N leaderboards and rankings in ClickHouse using materialized views that pre-aggregate scores without scanning billions of raw events.

---

## The Top-N Tracking Challenge

Finding the top 10 products by sales, top 100 users by engagement, or top 20 pages by views requires expensive aggregations over raw event tables. With billions of events, this can take seconds per query. Materialized views pre-aggregate these metrics so Top-N queries return instantly.

## Base Events Table

```sql
CREATE TABLE product_events
(
    event_time DateTime,
    product_id UInt64,
    category LowCardinality(String),
    event_type LowCardinality(String),
    revenue Decimal(10, 2),
    user_id UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (category, event_time, product_id);
```

## Product Score Aggregation View

```sql
CREATE TABLE product_daily_scores
(
    score_date Date,
    product_id UInt64,
    category LowCardinality(String),
    view_count UInt64,
    purchase_count UInt64,
    revenue_sum Decimal(10, 2)
)
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(score_date)
ORDER BY (score_date, category, product_id);

CREATE MATERIALIZED VIEW product_daily_scores_mv
TO product_daily_scores
AS
SELECT
    toDate(event_time) AS score_date,
    product_id,
    category,
    countIf(event_type = 'view') AS view_count,
    countIf(event_type = 'purchase') AS purchase_count,
    sumIf(revenue, event_type = 'purchase') AS revenue_sum
FROM product_events
GROUP BY score_date, product_id, category;
```

## Query Top-N by Revenue

```sql
-- Top 10 products by revenue in last 7 days
SELECT
    product_id,
    category,
    sum(revenue_sum) AS total_revenue,
    sum(purchase_count) AS total_purchases,
    sum(view_count) AS total_views,
    round(sum(purchase_count) / nullIf(sum(view_count), 0) * 100, 2) AS conversion_pct
FROM product_daily_scores
WHERE score_date >= today() - 7
GROUP BY product_id, category
ORDER BY total_revenue DESC
LIMIT 10;
```

## Category-Level Top-N

```sql
-- Top 5 products per category
SELECT *
FROM (
    SELECT
        product_id,
        category,
        sum(revenue_sum) AS total_revenue,
        row_number() OVER (PARTITION BY category ORDER BY sum(revenue_sum) DESC) AS rank_in_category
    FROM product_daily_scores
    WHERE score_date >= today() - 30
    GROUP BY product_id, category
)
WHERE rank_in_category <= 5
ORDER BY category, rank_in_category;
```

## Real-Time Leaderboard with Backfill

When a materialized view is first created, it only processes new data. To backfill existing data:

```sql
-- Backfill historical data into the score table
INSERT INTO product_daily_scores
SELECT
    toDate(event_time) AS score_date,
    product_id,
    category,
    countIf(event_type = 'view') AS view_count,
    countIf(event_type = 'purchase') AS purchase_count,
    sumIf(revenue, event_type = 'purchase') AS revenue_sum
FROM product_events
WHERE event_time < now()
GROUP BY score_date, product_id, category;
```

## Hourly Trending Products

```sql
CREATE TABLE product_hourly_scores
(
    score_hour DateTime,
    product_id UInt64,
    event_count UInt64
)
ENGINE = SummingMergeTree
ORDER BY (score_hour, product_id);

CREATE MATERIALIZED VIEW product_hourly_mv
TO product_hourly_scores
AS
SELECT
    toStartOfHour(event_time) AS score_hour,
    product_id,
    count() AS event_count
FROM product_events
GROUP BY score_hour, product_id;
```

## Summary

Top-N tracking in ClickHouse is best implemented with SummingMergeTree target tables and materialized views that aggregate scores at insert time. Daily or hourly score tables enable instant leaderboard queries without scanning raw event data. Use window functions like `row_number()` for per-category rankings on top of the pre-aggregated scores.
