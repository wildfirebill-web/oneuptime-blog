# How to Implement UTM Parameter Tracking in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UTM, Analytics, Marketing, Attribution

Description: Learn how to store, query, and analyze UTM parameters in ClickHouse to understand which marketing campaigns drive traffic and conversions.

---

UTM parameters are the backbone of marketing attribution. Capturing them in ClickHouse lets you run fast campaign performance queries across millions of sessions without hitting the limits of a traditional OLTP database.

## Table Design for UTM Tracking

Store each session with its UTM values alongside the user and timestamp.

```sql
CREATE TABLE utm_sessions
(
    session_id UUID DEFAULT generateUUIDv4(),
    user_id UInt64,
    utm_source LowCardinality(String),
    utm_medium LowCardinality(String),
    utm_campaign String,
    utm_term String,
    utm_content String,
    landing_page String,
    session_start DateTime,
    converted UInt8 DEFAULT 0,
    revenue Decimal(10, 2) DEFAULT 0
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(session_start)
ORDER BY (utm_source, utm_medium, session_start);
```

## Top Sources by Session Count

```sql
SELECT
    utm_source,
    utm_medium,
    count() AS sessions,
    countIf(converted = 1) AS conversions,
    round(countIf(converted = 1) * 100.0 / count(), 2) AS cvr_pct,
    sum(revenue) AS total_revenue
FROM utm_sessions
WHERE session_start >= now() - INTERVAL 30 DAY
GROUP BY utm_source, utm_medium
ORDER BY sessions DESC
LIMIT 20;
```

## Campaign Performance Comparison

Compare multiple campaigns side by side.

```sql
SELECT
    utm_campaign,
    utm_source,
    count() AS sessions,
    countDistinct(user_id) AS unique_users,
    sum(revenue) AS revenue,
    round(sum(revenue) / nullIf(count(), 0), 2) AS rps
FROM utm_sessions
WHERE session_start BETWEEN '2026-01-01' AND '2026-03-31'
  AND utm_medium = 'cpc'
GROUP BY utm_campaign, utm_source
ORDER BY revenue DESC;
```

## First-Touch Attribution

Assign revenue to the first UTM source a user ever touched.

```sql
WITH first_touch AS (
    SELECT
        user_id,
        argMin(utm_source, session_start) AS first_source,
        argMin(utm_campaign, session_start) AS first_campaign
    FROM utm_sessions
    GROUP BY user_id
)
SELECT
    ft.first_source,
    ft.first_campaign,
    count() AS users,
    sum(s.revenue) AS attributed_revenue
FROM utm_sessions s
JOIN first_touch ft ON s.user_id = ft.user_id
GROUP BY ft.first_source, ft.first_campaign
ORDER BY attributed_revenue DESC
LIMIT 15;
```

## Daily Trend by Medium

Track how paid vs organic traffic evolves over time.

```sql
SELECT
    toDate(session_start) AS day,
    utm_medium,
    count() AS sessions,
    sum(revenue) AS revenue
FROM utm_sessions
WHERE session_start >= now() - INTERVAL 90 DAY
GROUP BY day, utm_medium
ORDER BY day ASC, utm_medium;
```

## Materialized View for Daily Campaign Stats

```sql
CREATE MATERIALIZED VIEW utm_campaign_daily
ENGINE = SummingMergeTree()
ORDER BY (utm_source, utm_medium, utm_campaign, day)
AS
SELECT
    utm_source,
    utm_medium,
    utm_campaign,
    toDate(session_start) AS day,
    count() AS sessions,
    sum(converted) AS conversions,
    sum(revenue) AS revenue
FROM utm_sessions
GROUP BY utm_source, utm_medium, utm_campaign, day;
```

## Summary

Storing UTM data in ClickHouse with `LowCardinality` columns for source and medium keeps storage compact while enabling fast GROUP BY queries. `argMin` gives you first-touch attribution without expensive self-joins, and materialized views let dashboards read pre-aggregated campaign stats instantly. This approach scales to tens of millions of sessions per day with sub-second query times.
