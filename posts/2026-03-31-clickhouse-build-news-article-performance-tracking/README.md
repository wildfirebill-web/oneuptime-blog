# How to Build News Article Performance Tracking with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, News Analytics, Article Performance, Page View, Recirculation

Description: Learn how to track news article performance metrics including page views, time on page, recirculation, and traffic source attribution in ClickHouse.

---

News organizations need to understand article performance in real time - which stories are trending, where readers come from, and how long they stay. ClickHouse supports the high write throughput required during breaking news events while keeping analytical queries fast for editorial dashboards.

## Schema

```sql
CREATE TABLE article_pageviews
(
    ts              DateTime64(3),
    article_id      UInt64,
    section         LowCardinality(String),
    user_id         UInt64,
    session_id      UUID,
    is_subscriber   UInt8,
    traffic_source  LowCardinality(String), -- 'direct','search','social','email','push'
    time_on_page_s  UInt32,
    country_code    LowCardinality(FixedString(2))
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (article_id, ts);
```

## Real-Time Trending Articles

```sql
SELECT
    article_id,
    count() AS views_last_5min
FROM article_pageviews
WHERE ts >= now() - INTERVAL 5 MINUTE
GROUP BY article_id
ORDER BY views_last_5min DESC
LIMIT 10;
```

## Time on Page Distribution

```sql
SELECT
    multiIf(
        time_on_page_s < 10,  'bounce',
        time_on_page_s < 60,  'skim',
        time_on_page_s < 300, 'read',
        'deep_read'
    ) AS read_category,
    count()  AS readers,
    count() * 100.0 / sum(count()) OVER () AS pct
FROM article_pageviews
WHERE article_id = 9001
GROUP BY read_category;
```

## Traffic Source Mix

```sql
SELECT
    traffic_source,
    count() AS views,
    uniqExact(session_id) AS sessions
FROM article_pageviews
WHERE ts >= toStartOfDay(now())
GROUP BY traffic_source
ORDER BY views DESC;
```

## Subscriber vs Non-Subscriber Engagement

```sql
SELECT
    is_subscriber,
    avg(time_on_page_s) AS avg_time_on_page,
    count()             AS views
FROM article_pageviews
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY is_subscriber;
```

## Recirculation Rate

How often do readers view more than one article per session?

```sql
SELECT
    uniqExactIf(session_id, articles_per_session > 1) * 100.0 /
        uniqExact(session_id) AS recirculation_rate
FROM (
    SELECT session_id, uniqExact(article_id) AS articles_per_session
    FROM article_pageviews
    WHERE ts >= now() - INTERVAL 1 DAY
    GROUP BY session_id
);
```

## Section Performance

```sql
SELECT
    section,
    count()              AS total_views,
    avg(time_on_page_s)  AS avg_time_s,
    uniqExact(user_id)   AS unique_readers
FROM article_pageviews
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY section
ORDER BY total_views DESC;
```

## Summary

ClickHouse provides the throughput and query flexibility that news analytics demands. Track trending articles in real time, segment readers by traffic source and subscription status, and measure recirculation rates to understand reader loyalty. Partition by day and pre-aggregate hourly summaries with materialized views to support fast editorial dashboards during peak news events.
