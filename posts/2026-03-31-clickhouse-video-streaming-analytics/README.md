# How to Build Video Streaming Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Video Streaming, Analytics, QoE, Rebuffering, CDN

Description: Build video streaming analytics in ClickHouse to track Quality of Experience metrics like rebuffering rates, bitrate, startup time, and CDN performance.

---

Video streaming platforms need to monitor Quality of Experience (QoE) in real time - rebuffering events, adaptive bitrate switches, startup failures, and CDN performance all directly impact viewer satisfaction and churn. ClickHouse handles the high event volume from streaming sessions.

## Streaming Events Table

```sql
CREATE TABLE streaming_events (
    session_id   String,
    user_id      UInt64,
    device_type  LowCardinality(String),  -- mobile, tv, desktop, tablet
    platform     LowCardinality(String),  -- ios, android, web, roku
    content_id   String,
    content_type LowCardinality(String),  -- vod, live
    cdn_node     LowCardinality(String),
    region       LowCardinality(String),
    event_type   LowCardinality(String),  -- start, play, pause, buffer, bitrate_change, error, end
    bitrate_kbps UInt32,
    buffer_ms    UInt32,  -- buffering duration for buffer events
    startup_ms   UInt32,  -- startup latency for start events
    error_code   Nullable(String),
    session_duration_s UInt32,
    recorded_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (region, device_type, recorded_at);
```

## Rebuffering Rate by CDN and Region

```sql
SELECT
    cdn_node,
    region,
    countDistinct(session_id) AS total_sessions,
    countDistinctIf(session_id, event_type = 'buffer') AS sessions_with_buffering,
    round(countDistinctIf(session_id, event_type = 'buffer') /
        countDistinct(session_id) * 100, 2) AS rebuffering_rate_pct,
    avgIf(buffer_ms, event_type = 'buffer') AS avg_buffer_duration_ms
FROM streaming_events
WHERE recorded_at >= today() - 1
GROUP BY cdn_node, region
ORDER BY rebuffering_rate_pct DESC;
```

## Startup Time Distribution

```sql
SELECT
    platform,
    device_type,
    avg(startup_ms) AS avg_startup_ms,
    quantile(0.5)(startup_ms) AS median_startup_ms,
    quantile(0.95)(startup_ms) AS p95_startup_ms,
    countIf(startup_ms > 3000) AS slow_starts,
    count() AS total_starts
FROM streaming_events
WHERE event_type = 'start'
  AND recorded_at >= today() - 7
GROUP BY platform, device_type
ORDER BY p95_startup_ms DESC;
```

## Bitrate Distribution

Understand what quality tiers users experience:

```sql
SELECT
    device_type,
    multiIf(
        bitrate_kbps < 500, 'Low (<500kbps)',
        bitrate_kbps < 2000, 'Medium (500-2000kbps)',
        bitrate_kbps < 5000, 'High (2-5Mbps)',
        'HD (>5Mbps)'
    ) AS quality_tier,
    count() AS event_count,
    round(count() / sum(count()) OVER (PARTITION BY device_type) * 100, 2) AS pct_of_playback
FROM streaming_events
WHERE event_type = 'play'
  AND recorded_at >= today() - 7
GROUP BY device_type, quality_tier
ORDER BY device_type, event_count DESC;
```

## Error Rate by Platform

Track streaming errors to identify platform-specific issues:

```sql
SELECT
    platform,
    error_code,
    count() AS error_count,
    countDistinct(session_id) AS affected_sessions,
    round(count() / sum(count()) OVER (PARTITION BY platform) * 100, 2) AS error_share_pct
FROM streaming_events
WHERE event_type = 'error'
  AND recorded_at >= today() - 7
GROUP BY platform, error_code
ORDER BY platform, error_count DESC;
```

## Session Quality Score (MOS Proxy)

Compute a composite quality score per session:

```sql
SELECT
    session_id,
    user_id,
    device_type,
    platform,
    region,
    session_duration_s,
    countIf(event_type = 'buffer') AS buffer_events,
    sumIf(buffer_ms, event_type = 'buffer') AS total_buffer_ms,
    avg(bitrate_kbps) AS avg_bitrate,
    -- simple quality score: penalize buffering and low bitrate
    greatest(0, 100
        - countIf(event_type = 'buffer') * 10
        - sumIf(buffer_ms, event_type = 'buffer') / 1000 * 5
        - if(avg(bitrate_kbps) < 500, 30, if(avg(bitrate_kbps) < 2000, 10, 0))
    ) AS quality_score
FROM streaming_events
WHERE recorded_at >= today() - 1
GROUP BY session_id, user_id, device_type, platform, region, session_duration_s
ORDER BY quality_score;
```

## Summary

ClickHouse is purpose-built for streaming analytics - handling billions of player events daily to power real-time QoE dashboards. Rebuffering rates, startup time distributions, bitrate analysis, and error tracking give streaming teams the visibility they need to maintain a high-quality viewer experience.
