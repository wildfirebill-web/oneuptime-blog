# How to Calculate Bounce Rate in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bounce Rate, Web Analytics, SQL, Metric

Description: Calculate website bounce rate in ClickHouse by identifying single-page sessions and expressing them as a percentage of all sessions.

---

## What Is Bounce Rate

Bounce rate = sessions with only one page view / total sessions * 100.

A session "bounces" when the visitor leaves without viewing a second page.

## Table Schema

```sql
CREATE TABLE page_views (
    session_id  String,
    page_path   String,
    user_id     UInt64,
    ts          DateTime
) ENGINE = MergeTree()
ORDER BY (ts, session_id);
```

## Overall Bounce Rate

```sql
WITH session_counts AS (
    SELECT
        session_id,
        count() AS page_count
    FROM page_views
    WHERE ts >= today() - 30
    GROUP BY session_id
)
SELECT
    countIf(page_count = 1)                     AS bounced_sessions,
    count()                                     AS total_sessions,
    round(countIf(page_count = 1) * 100.0 / count(), 2) AS bounce_rate_pct
FROM session_counts;
```

## Bounce Rate by Landing Page

```sql
WITH session_first_page AS (
    SELECT
        session_id,
        argMin(page_path, ts)   AS landing_page,
        count()                 AS page_count
    FROM page_views
    WHERE ts >= today() - 30
    GROUP BY session_id
)
SELECT
    landing_page,
    countIf(page_count = 1)                                  AS bounces,
    count()                                                  AS sessions,
    round(countIf(page_count = 1) * 100.0 / count(), 2)     AS bounce_rate_pct
FROM session_first_page
GROUP BY landing_page
HAVING sessions >= 50
ORDER BY bounce_rate_pct DESC
LIMIT 20;
```

## Daily Bounce Rate Trend

```sql
WITH session_counts AS (
    SELECT
        session_id,
        toDate(min(ts))  AS day,
        count()          AS page_count
    FROM page_views
    WHERE ts >= today() - 30
    GROUP BY session_id
)
SELECT
    day,
    round(countIf(page_count = 1) * 100.0 / count(), 2) AS bounce_rate_pct,
    count()                                              AS sessions
FROM session_counts
GROUP BY day
ORDER BY day;
```

## Bounce Rate by Traffic Source

If you track UTM parameters:

```sql
WITH session_meta AS (
    SELECT
        session_id,
        argMin(utm_source, ts) AS source,
        count()                AS page_count
    FROM page_views
    WHERE ts >= today() - 30
    GROUP BY session_id
)
SELECT
    source,
    round(countIf(page_count = 1) * 100.0 / count(), 2) AS bounce_rate_pct,
    count()                                              AS sessions
FROM session_meta
GROUP BY source
ORDER BY sessions DESC;
```

## Filtering Bot Traffic

```sql
WHERE ts >= today() - 30
  AND user_agent NOT LIKE '%bot%'
  AND user_agent NOT LIKE '%crawler%'
```

## Summary

ClickHouse calculates bounce rate efficiently using CTEs that count page views per session, then aggregate with `countIf(page_count = 1)`. Break down by landing page, day, or traffic source to identify which pages and channels drive the highest bounce rates.
