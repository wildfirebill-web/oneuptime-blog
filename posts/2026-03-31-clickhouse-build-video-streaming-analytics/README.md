# How to Build Video Streaming Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Video Streaming, Analytics, Media, Time Series, CDN

Description: Build a video streaming analytics platform in ClickHouse to track playback events, buffering ratios, viewer engagement, and content performance.

---

## Introduction

Video streaming platforms generate enormous volumes of playback events - play, pause, buffer, seek, and quality change - from every viewer session. ClickHouse stores these events efficiently and enables the fast aggregations needed to measure viewer experience, content performance, and infrastructure quality in real time.

## Schema

```sql
CREATE TABLE playback_events
(
    session_id     String,
    user_id        UInt64,
    content_id     UInt64,
    cdn_node       LowCardinality(String),
    event_type     LowCardinality(String),
    occurred_at    DateTime64(3),
    quality_p      UInt16,
    bitrate_kbps   UInt32,
    buffer_ms      UInt32,
    latency_ms     UInt16,
    country_code   LowCardinality(FixedString(2)),
    device_type    LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(occurred_at)
ORDER BY (content_id, occurred_at)
TTL toDate(occurred_at) + INTERVAL 1 YEAR;
```

## Buffering Ratio by Content

```sql
SELECT
    content_id,
    toDate(occurred_at) AS day,
    countIf(event_type = 'buffer_start')       AS buffer_events,
    countIf(event_type = 'play')               AS play_events,
    round(countIf(event_type = 'buffer_start') / countIf(event_type = 'play') * 100, 3) AS buffer_ratio_pct
FROM playback_events
WHERE occurred_at >= today() - INTERVAL 7 DAY
GROUP BY content_id, day
ORDER BY buffer_ratio_pct DESC
LIMIT 30;
```

## Average Bitrate by CDN Node

```sql
SELECT
    cdn_node,
    toStartOfHour(occurred_at) AS hour,
    avg(bitrate_kbps)          AS avg_bitrate_kbps,
    quantile(0.05)(bitrate_kbps) AS p5_bitrate,
    count()                    AS events
FROM playback_events
WHERE event_type = 'quality_change'
  AND occurred_at >= now() - INTERVAL 24 HOUR
GROUP BY cdn_node, hour
ORDER BY cdn_node, hour;
```

## Viewer Engagement - Watch Duration

```sql
WITH session_stats AS (
    SELECT
        session_id,
        content_id,
        user_id,
        min(occurred_at) AS session_start,
        max(occurred_at) AS session_end,
        dateDiff('second', min(occurred_at), max(occurred_at)) AS watch_seconds
    FROM playback_events
    WHERE occurred_at >= today() - INTERVAL 7 DAY
    GROUP BY session_id, content_id, user_id
)
SELECT
    content_id,
    count(DISTINCT session_id)         AS total_sessions,
    round(avg(watch_seconds) / 60, 1) AS avg_watch_min,
    round(quantile(0.5)(watch_seconds) / 60, 1) AS median_watch_min
FROM session_stats
GROUP BY content_id
ORDER BY avg_watch_min DESC
LIMIT 20;
```

## Real-Time Error Rate Dashboard

```sql
SELECT
    toStartOfFiveMinutes(occurred_at) AS interval_start,
    cdn_node,
    countIf(event_type = 'error')     AS errors,
    countIf(event_type = 'play')      AS plays,
    round(countIf(event_type = 'error') / countIf(event_type = 'play') * 100, 4) AS error_rate_pct
FROM playback_events
WHERE occurred_at >= now() - INTERVAL 1 HOUR
GROUP BY interval_start, cdn_node
ORDER BY interval_start DESC, error_rate_pct DESC;
```

## Top Content by Unique Viewers

```sql
SELECT
    content_id,
    toDate(occurred_at)          AS day,
    uniq(user_id)                AS unique_viewers,
    count(DISTINCT session_id)   AS total_sessions
FROM playback_events
WHERE event_type = 'play'
  AND occurred_at >= today() - INTERVAL 30 DAY
GROUP BY content_id, day
ORDER BY unique_viewers DESC
LIMIT 20;
```

## Quality Distribution by Device Type

```sql
SELECT
    device_type,
    toYYYYMM(occurred_at)         AS month,
    countIf(quality_p >= 1080)    AS hd_plus_sessions,
    countIf(quality_p = 720)      AS hd_sessions,
    countIf(quality_p < 720)      AS sd_sessions,
    count()                       AS total_sessions
FROM playback_events
WHERE event_type = 'play'
  AND occurred_at >= today() - INTERVAL 90 DAY
GROUP BY device_type, month
ORDER BY device_type, month;
```

## Summary

ClickHouse is an excellent backbone for video streaming analytics, handling billions of playback events with fast query response. Buffering ratio monitoring, viewer engagement, error rate dashboards, and quality distribution analysis all run efficiently without pre-aggregation, while materialized views can speed up frequently queried dashboards further.
