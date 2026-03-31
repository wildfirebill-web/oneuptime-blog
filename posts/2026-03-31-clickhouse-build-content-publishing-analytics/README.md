# How to Build Content Publishing Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Content Analytics, Publishing, Page View, Engagement, Analytics

Description: Learn how to track article views, scroll depth, shares, and author performance metrics in ClickHouse for a content publishing platform.

---

Content publishing platforms need to measure how articles perform over time - from the first click to the final scroll. ClickHouse enables high-throughput event ingestion and flexible querying so editorial and product teams can answer questions like "which articles drive return visits?" and "how does engagement decay after publishing?"

## Event Schema

```sql
CREATE TABLE content_events
(
    ts           DateTime64(3),
    article_id   UInt64,
    author_id    UInt64,
    user_id      UInt64,
    session_id   UUID,
    event_type   LowCardinality(String), -- 'view','scroll','share','comment','like'
    scroll_pct   UInt8,                  -- 0-100
    referrer     LowCardinality(String),
    country_code LowCardinality(FixedString(2))
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (article_id, ts);
```

## Top Articles by Unique Readers

```sql
SELECT
    article_id,
    uniqExact(user_id) AS unique_readers,
    count()            AS total_views
FROM content_events
WHERE event_type = 'view'
  AND ts >= now() - INTERVAL 7 DAY
GROUP BY article_id
ORDER BY unique_readers DESC
LIMIT 20;
```

## Average Scroll Depth per Article

```sql
SELECT
    article_id,
    avg(scroll_pct) AS avg_scroll_depth,
    countIf(scroll_pct >= 80) * 100.0 / count() AS pct_completed
FROM content_events
WHERE event_type = 'scroll'
  AND ts >= now() - INTERVAL 30 DAY
GROUP BY article_id
ORDER BY avg_scroll_depth DESC
LIMIT 20;
```

## Engagement Decay Over Days Since Publication

```sql
WITH article_publish_dates AS (
    SELECT article_id, min(ts) AS published_at
    FROM content_events
    WHERE event_type = 'view'
    GROUP BY article_id
)
SELECT
    dateDiff('day', apd.published_at, e.ts) AS days_since_publish,
    count() AS views
FROM content_events e
JOIN article_publish_dates apd USING (article_id)
WHERE e.event_type = 'view'
  AND days_since_publish BETWEEN 0 AND 30
GROUP BY days_since_publish
ORDER BY days_since_publish;
```

## Author Performance Dashboard

```sql
SELECT
    author_id,
    uniqExact(article_id)  AS articles_published,
    countIf(event_type = 'view') AS total_views,
    countIf(event_type = 'share') AS total_shares,
    countIf(event_type = 'share') * 100.0 /
        countIf(event_type = 'view') AS share_rate_pct
FROM content_events
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY author_id
ORDER BY total_views DESC
LIMIT 20;
```

## Referrer Attribution

```sql
SELECT
    referrer,
    uniqExact(session_id) AS sessions,
    countIf(event_type = 'share') AS shares
FROM content_events
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY referrer
ORDER BY sessions DESC
LIMIT 15;
```

## Materialized View for Daily Article Stats

```sql
CREATE MATERIALIZED VIEW article_daily_stats_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(day)
ORDER BY (day, article_id)
AS
SELECT
    toDate(ts)   AS day,
    article_id,
    countIf(event_type = 'view')    AS views,
    countIf(event_type = 'share')   AS shares,
    countIf(event_type = 'comment') AS comments,
    uniqExactState(user_id)         AS unique_users
FROM content_events
GROUP BY day, article_id;
```

## Summary

ClickHouse supports content publishing analytics at scale by combining fast event ingestion with expressive aggregation queries. Track unique readers, scroll depth, referrer attribution, and author performance using a single events table. Pre-aggregate daily statistics with materialized views to keep dashboard queries fast even as your article catalog grows to millions of posts.
