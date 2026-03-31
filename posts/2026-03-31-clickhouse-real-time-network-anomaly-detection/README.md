# How to Build Real-Time Network Anomaly Detection with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Network Monitoring, Anomaly Detection, Telecom, Real-Time Analytics

Description: Detect network anomalies such as traffic spikes, latency degradation, and link failures in real time using ClickHouse statistical queries.

---

Network anomalies - sudden traffic spikes, packet loss surges, and link failures - can cascade into outages if not caught quickly. ClickHouse can ingest telemetry at high rates and run statistical detection queries in seconds, making it a practical real-time anomaly detection engine.

## Network Telemetry Table

```sql
CREATE TABLE network_telemetry (
    device_id       UInt32,
    interface       LowCardinality(String),
    region          LowCardinality(String),
    sampled_at      DateTime,
    bytes_in        UInt64,
    bytes_out       UInt64,
    packets_in      UInt64,
    packets_out     UInt64,
    errors_in       UInt32,
    errors_out      UInt32,
    drops_in        UInt32,
    drops_out       UInt32,
    latency_ms      Float32,
    link_up         UInt8
) ENGINE = MergeTree()
ORDER BY (device_id, interface, sampled_at)
PARTITION BY toYYYYMM(sampled_at);
```

## Baseline Computation (Last 7 Days)

```sql
CREATE TABLE IF NOT EXISTS baseline_stats
ENGINE = Memory
AS
SELECT
    device_id,
    interface,
    toHour(sampled_at)             AS hour_of_day,
    avg(bytes_in)                  AS mean_bytes_in,
    stddevPop(bytes_in)            AS std_bytes_in,
    avg(latency_ms)                AS mean_latency,
    stddevPop(latency_ms)          AS std_latency
FROM network_telemetry
WHERE sampled_at >= now() - INTERVAL 7 DAY
  AND sampled_at < now() - INTERVAL 5 MINUTE
GROUP BY device_id, interface, hour_of_day;
```

## Z-Score Based Traffic Spike Detection

```sql
SELECT
    t.device_id,
    t.interface,
    t.sampled_at,
    t.bytes_in,
    b.mean_bytes_in,
    round((t.bytes_in - b.mean_bytes_in) / nullIf(b.std_bytes_in, 0), 2) AS z_score
FROM network_telemetry t
JOIN baseline_stats b
    ON t.device_id = b.device_id
   AND t.interface = b.interface
   AND toHour(t.sampled_at) = b.hour_of_day
WHERE t.sampled_at >= now() - INTERVAL 10 MINUTE
  AND abs((t.bytes_in - b.mean_bytes_in) / nullIf(b.std_bytes_in, 0)) > 3
ORDER BY abs(z_score) DESC
LIMIT 20;
```

A Z-score above 3 indicates the current traffic is more than 3 standard deviations from the hourly baseline.

## Detecting Link Flapping

```sql
SELECT
    device_id,
    interface,
    count()        AS state_changes
FROM (
    SELECT
        device_id,
        interface,
        sampled_at,
        link_up,
        lagInFrame(link_up) OVER (PARTITION BY device_id, interface ORDER BY sampled_at) AS prev_state
    FROM network_telemetry
    WHERE sampled_at >= now() - INTERVAL 1 HOUR
)
WHERE link_up != prev_state
GROUP BY device_id, interface
HAVING state_changes > 3
ORDER BY state_changes DESC;
```

## Error Rate Spike Detection

```sql
SELECT
    device_id,
    interface,
    sampled_at,
    errors_in + errors_out          AS total_errors,
    packets_in + packets_out        AS total_packets,
    round(100.0 * (errors_in + errors_out) / nullIf(packets_in + packets_out, 0), 3) AS error_rate_pct
FROM network_telemetry
WHERE sampled_at >= now() - INTERVAL 15 MINUTE
  AND (errors_in + errors_out) > 0
HAVING error_rate_pct > 1.0
ORDER BY error_rate_pct DESC
LIMIT 20;
```

## Alerting Integration with OneUptime

Push anomaly results to OneUptime via its API on a scheduled basis. Configure OneUptime monitors to watch a ClickHouse-backed endpoint and trigger incidents when anomaly counts exceed thresholds, enabling automated on-call alerting without a separate AIOps platform.

## Summary

ClickHouse enables network anomaly detection by storing high-resolution telemetry and computing Z-score deviations from historical baselines in real time. Window functions detect link flapping, conditional aggregation surfaces error spikes, and the whole pipeline runs without external stream processors - just ClickHouse SQL and a scheduler.
