# How to Analyze VoIP Quality Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, VoIP, Telecom, Quality of Service, MOS Score

Description: Store and analyze VoIP session quality metrics in ClickHouse to track MOS scores, jitter, packet loss, and identify degraded call routes.

---

VoIP quality is invisible to users until it fails. Choppy audio, dropped calls, and echo are symptoms of network problems that can be quantified through metrics like MOS scores, jitter, and packet loss. ClickHouse lets you analyze these metrics across millions of concurrent calls.

## VoIP Session Metrics Table

```sql
CREATE TABLE voip_sessions (
    session_id      UUID,
    caller_id       UInt64,
    callee_id       UInt64,
    start_time      DateTime,
    end_time        DateTime,
    duration_secs   UInt32,
    codec           LowCardinality(String),   -- 'G.711', 'G.729', 'Opus'
    direction       LowCardinality(String),
    route           LowCardinality(String),
    mos_score       Float32,    -- Mean Opinion Score 1.0-5.0
    jitter_ms       Float32,
    packet_loss_pct Float32,
    rtt_ms          Float32,    -- round-trip time
    r_factor        Float32,    -- E-model R-factor
    burst_loss_rate Float32,
    end_cause       LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (route, start_time)
PARTITION BY toYYYYMM(start_time);
```

## Average MOS Score by Route

```sql
SELECT
    route,
    round(avg(mos_score), 3)          AS avg_mos,
    round(quantile(0.05)(mos_score), 3) AS p5_mos,
    count()                           AS sessions
FROM voip_sessions
WHERE start_time >= today() - 7
GROUP BY route
ORDER BY avg_mos ASC
LIMIT 20;
```

Routes with average MOS below 3.5 indicate poor call quality that needs investigation.

## MOS Score Distribution

```sql
SELECT
    multiIf(
        mos_score >= 4.3, 'excellent',
        mos_score >= 4.0, 'good',
        mos_score >= 3.6, 'fair',
        mos_score >= 3.1, 'poor',
        'bad'
    ) AS quality_tier,
    count()                                              AS sessions,
    round(100.0 * count() / sum(count()) OVER (), 2)    AS pct
FROM voip_sessions
WHERE start_time >= today() - 7
GROUP BY quality_tier
ORDER BY sessions DESC;
```

## Packet Loss Impact on MOS

```sql
SELECT
    round(packet_loss_pct, 1) AS loss_bucket,
    round(avg(mos_score), 3)  AS avg_mos,
    count()                   AS samples
FROM voip_sessions
WHERE start_time >= today() - 7
  AND packet_loss_pct <= 20
GROUP BY loss_bucket
ORDER BY loss_bucket;
```

## Codec Quality Comparison

```sql
SELECT
    codec,
    round(avg(mos_score), 3)          AS avg_mos,
    round(avg(jitter_ms), 2)          AS avg_jitter_ms,
    round(avg(packet_loss_pct), 3)    AS avg_loss_pct,
    count()                           AS sessions
FROM voip_sessions
WHERE start_time >= today() - 30
GROUP BY codec
ORDER BY avg_mos DESC;
```

## Hourly Quality Degradation Pattern

```sql
SELECT
    toStartOfHour(start_time)       AS hour,
    round(avg(mos_score), 3)        AS avg_mos,
    round(avg(packet_loss_pct), 3)  AS avg_loss,
    count()                         AS calls
FROM voip_sessions
WHERE start_time >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;
```

## Routes with Persistent Quality Issues

```sql
SELECT
    route,
    countIf(mos_score < 3.5)                            AS poor_sessions,
    count()                                              AS total_sessions,
    round(100.0 * countIf(mos_score < 3.5) / count(), 2) AS poor_pct
FROM voip_sessions
WHERE start_time >= today() - 7
GROUP BY route
HAVING poor_pct > 10
ORDER BY poor_pct DESC;
```

## Summary

ClickHouse is well-suited for VoIP quality analytics because its columnar storage efficiently compresses repetitive codec and route fields, and its aggregation functions provide the statistical depth needed for quality analysis. By tracking MOS scores, jitter, and packet loss over time, operations teams can pinpoint degraded routes and codec issues before they impact customers.
