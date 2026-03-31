# How to Monitor Game Server Performance with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Game Server, Performance Monitoring, Latency, Gaming Infrastructure

Description: Monitor game server performance metrics with ClickHouse to track tick rate, latency, CPU usage, and player connection quality.

---

Game server performance directly affects player experience. High tick rates, low latency, and stable CPU usage are critical for competitive gameplay. ClickHouse stores and analyzes server telemetry at the granularity needed to diagnose issues and optimize infrastructure.

## Server Metrics Table

```sql
CREATE TABLE game_server_metrics (
    timestamp       DateTime,
    server_id       String,
    region          LowCardinality(String),
    game_mode       LowCardinality(String),
    tick_rate       Float32,
    cpu_pct         Float32,
    mem_pct         Float32,
    player_count    UInt16,
    avg_latency_ms  Float32,
    p95_latency_ms  Float32,
    packet_loss_pct Float32,
    date            Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (server_id, timestamp);
```

## Current Server Health

```sql
SELECT
    server_id,
    region,
    round(avg(tick_rate), 1) AS avg_tick_rate,
    round(avg(cpu_pct), 1) AS avg_cpu,
    round(avg(avg_latency_ms), 1) AS avg_latency,
    round(avg(packet_loss_pct), 3) AS avg_packet_loss,
    max(player_count) AS peak_players
FROM game_server_metrics
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY server_id, region
ORDER BY avg_latency DESC;
```

## Tick Rate Degradation Detection

Flag servers with tick rate below acceptable threshold:

```sql
SELECT
    server_id,
    region,
    count() AS degraded_samples,
    round(avg(tick_rate), 1) AS avg_tick_rate,
    min(tick_rate) AS min_tick_rate
FROM game_server_metrics
WHERE timestamp >= now() - INTERVAL 30 MINUTE
  AND tick_rate < 60
GROUP BY server_id, region
ORDER BY degraded_samples DESC;
```

## Latency Percentiles by Region

```sql
SELECT
    region,
    quantile(0.50)(avg_latency_ms) AS p50_latency,
    quantile(0.95)(p95_latency_ms) AS p95_latency,
    quantile(0.99)(p95_latency_ms) AS p99_latency
FROM game_server_metrics
WHERE date = today()
GROUP BY region
ORDER BY p95_latency DESC;
```

## Correlation: Player Count vs Latency

```sql
SELECT
    multiIf(player_count < 10, '1-9', player_count < 20, '10-19', player_count < 30, '20-29', '30+') AS player_bucket,
    round(avg(avg_latency_ms), 2) AS avg_latency,
    round(avg(cpu_pct), 1) AS avg_cpu
FROM game_server_metrics
WHERE date >= today() - 7
GROUP BY player_bucket
ORDER BY player_bucket;
```

## Peak Hours by Region

```sql
SELECT
    region,
    toHour(timestamp) AS hour_of_day,
    round(avg(player_count), 0) AS avg_players
FROM game_server_metrics
WHERE date >= today() - 7
GROUP BY region, hour_of_day
ORDER BY region, hour_of_day;
```

## Summary

ClickHouse stores dense server telemetry streams efficiently and delivers fast aggregations for latency percentiles, tick rate health, and player load analysis. By correlating server metrics with player counts and time of day, infrastructure teams can right-size capacity and pinpoint performance regressions quickly.
