# How to Track Content Consumption Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Content, Consumption, Analytics, Media, Engagement

Description: Track content consumption patterns in ClickHouse to understand what users read, watch, and share, and optimize content strategy with data-driven insights.

---

## Overview

Content consumption analytics answers questions like: what content drives the most engagement, when are users most active, which categories retain readers longest, and how does consumption change over time. ClickHouse stores user interaction events at scale and enables the SQL queries needed to answer these questions quickly.

## Schema

```sql
CREATE TABLE content_events
(
    event_id       UUID,
    user_id        UInt64,
    content_id     UInt64,
    category       LowCardinality(String),
    content_type   LowCardinality(String),
    event_type     LowCardinality(String),
    occurred_at    DateTime,
    duration_s     UInt32,
    scroll_pct     UInt8,
    country_code   LowCardinality(FixedString(2)),
    platform       LowCardinality(String),
    referrer       LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (content_id, occurred_at)
TTL occurred_at + INTERVAL 2 YEAR;
```

## Top Content by Engagement Time

```sql
SELECT
    content_id,
    content_type,
    count(DISTINCT user_id)   AS unique_consumers,
    count()                   AS total_events,
    round(avg(duration_s) / 60, 1) AS avg_time_min,
    round(avg(scroll_pct), 1) AS avg_scroll_pct
FROM content_events
WHERE event_type = 'view'
  AND occurred_at >= today() - INTERVAL 30 DAY
GROUP BY content_id, content_type
ORDER BY avg_time_min DESC
LIMIT 20;
```

## Hourly Consumption Heatmap

```sql
SELECT
    toDayOfWeek(occurred_at)   AS day_of_week,
    toHour(occurred_at)        AS hour_of_day,
    count()                    AS events,
    count(DISTINCT user_id)    AS unique_users
FROM content_events
WHERE event_type IN ('view', 'play')
  AND occurred_at >= today() - INTERVAL 90 DAY
GROUP BY day_of_week, hour_of_day
ORDER BY day_of_week, hour_of_day;
```

## Category Share of Total Consumption

```sql
SELECT
    category,
    toYYYYMM(occurred_at)                                     AS month,
    sum(duration_s)                                           AS total_seconds,
    round(sum(duration_s) * 100.0 / sum(sum(duration_s)) OVER (PARTITION BY toYYYYMM(occurred_at)), 2) AS share_pct
FROM content_events
WHERE event_type = 'view'
  AND occurred_at >= today() - INTERVAL 6 MONTH
GROUP BY category, month
ORDER BY month, share_pct DESC;
```

## Repeat Consumption Rate

Users who consumed the same content more than once:

```sql
SELECT
    content_id,
    count(DISTINCT user_id) AS unique_users,
    countIf(view_count > 1) AS repeat_viewers,
    round(countIf(view_count > 1) / count(DISTINCT user_id) * 100, 2) AS repeat_rate_pct
FROM (
    SELECT content_id, user_id, count() AS view_count
    FROM content_events
    WHERE event_type = 'view'
      AND occurred_at >= today() - INTERVAL 30 DAY
    GROUP BY content_id, user_id
) t
GROUP BY content_id
ORDER BY repeat_rate_pct DESC
LIMIT 20;
```

## Platform Comparison

```sql
SELECT
    platform,
    toYYYYMM(occurred_at)           AS month,
    count(DISTINCT user_id)         AS monthly_active_users,
    round(avg(duration_s) / 60, 1) AS avg_session_min,
    avg(scroll_pct)                 AS avg_scroll_pct
FROM content_events
WHERE event_type = 'view'
  AND occurred_at >= today() - INTERVAL 6 MONTH
GROUP BY platform, month
ORDER BY platform, month;
```

## Content Completion Rate

```sql
SELECT
    content_id,
    content_type,
    countIf(scroll_pct >= 80) AS completions,
    count()                    AS views,
    round(countIf(scroll_pct >= 80) / count() * 100, 2) AS completion_rate_pct
FROM content_events
WHERE event_type = 'view'
  AND occurred_at >= today() - INTERVAL 30 DAY
GROUP BY content_id, content_type
ORDER BY completion_rate_pct DESC
LIMIT 30;
```

## Summary

ClickHouse gives content teams a fast, scalable platform to analyze consumption patterns. Top content by engagement time, hourly heatmaps, category share trends, repeat consumption rates, and completion rates all run efficiently against billions of event rows. These insights inform editorial strategy, recommendation algorithms, and platform investment decisions.
