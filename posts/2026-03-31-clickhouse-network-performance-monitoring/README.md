# How to Build Network Performance Monitoring with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Network Monitoring, Telecom, Performance, Time Series

Description: Store and query network performance metrics in ClickHouse to track latency, packet loss, throughput, and SLA compliance across your infrastructure.

---

Network operations teams need to correlate performance metrics across thousands of devices in real time. ClickHouse's time-series query capabilities and high ingestion rate make it a strong choice for network performance monitoring (NPM) backends.

## Network Metrics Table

```sql
CREATE TABLE network_metrics (
    metric_id       UUID,
    device_id       UInt32,
    device_type     LowCardinality(String),  -- 'router', 'switch', 'cell_tower'
    interface       LowCardinality(String),
    region          LowCardinality(String),
    recorded_at     DateTime,
    latency_ms      Float32,
    packet_loss_pct Float32,
    jitter_ms       Float32,
    throughput_mbps Float32,
    error_count     UInt32,
    cpu_pct         Float32,
    memory_pct      Float32
) ENGINE = MergeTree()
ORDER BY (device_id, recorded_at)
PARTITION BY toYYYYMM(recorded_at)
TTL recorded_at + INTERVAL 90 DAY;
```

The `TTL` clause automatically drops data older than 90 days to keep storage lean.

## Average Latency by Region Over the Last Hour

```sql
SELECT
    region,
    round(avg(latency_ms), 2)      AS avg_latency_ms,
    round(max(latency_ms), 2)      AS max_latency_ms,
    round(quantile(0.95)(latency_ms), 2) AS p95_latency_ms
FROM network_metrics
WHERE recorded_at >= now() - INTERVAL 1 HOUR
GROUP BY region
ORDER BY avg_latency_ms DESC;
```

## SLA Compliance Check

Report the percentage of measurement intervals that stayed within the 10ms SLA.

```sql
SELECT
    device_id,
    round(100.0 * countIf(latency_ms <= 10) / count(), 2) AS sla_compliance_pct
FROM network_metrics
WHERE recorded_at >= today() - 30
GROUP BY device_id
HAVING sla_compliance_pct < 99.9
ORDER BY sla_compliance_pct ASC
LIMIT 20;
```

## Packet Loss Trend Over 24 Hours

```sql
SELECT
    toStartOfFiveMinutes(recorded_at) AS bucket,
    round(avg(packet_loss_pct), 3)    AS avg_loss_pct
FROM network_metrics
WHERE recorded_at >= now() - INTERVAL 24 HOUR
GROUP BY bucket
ORDER BY bucket;
```

## Top Devices by Error Count

```sql
SELECT
    device_id,
    device_type,
    sum(error_count) AS total_errors
FROM network_metrics
WHERE recorded_at >= today() - 7
GROUP BY device_id, device_type
ORDER BY total_errors DESC
LIMIT 10;
```

## Throughput Capacity Analysis

```sql
SELECT
    interface,
    round(avg(throughput_mbps), 1)       AS avg_mbps,
    round(max(throughput_mbps), 1)       AS peak_mbps,
    round(quantile(0.99)(throughput_mbps), 1) AS p99_mbps
FROM network_metrics
WHERE device_type = 'router'
  AND recorded_at >= today() - 7
GROUP BY interface
ORDER BY peak_mbps DESC;
```

## Materialized View for 5-Minute Rollups

```sql
CREATE MATERIALIZED VIEW network_5min_mv
ENGINE = AggregatingMergeTree()
ORDER BY (device_id, bucket)
AS
SELECT
    device_id,
    toStartOfFiveMinutes(recorded_at) AS bucket,
    avgState(latency_ms)              AS latency_state,
    avgState(packet_loss_pct)         AS loss_state,
    maxState(throughput_mbps)         AS peak_throughput_state
FROM network_metrics
GROUP BY device_id, bucket;
```

## Integrating with OneUptime

Connect OneUptime to your ClickHouse NPM database to create status pages and alerts. When packet loss exceeds threshold or a device's p95 latency spikes, OneUptime can page the on-call engineer automatically.

## Summary

ClickHouse provides the throughput and aggregation speed needed for real-time network performance monitoring. Use TTL policies to manage data lifecycle, `AggregatingMergeTree` for pre-computed rollups, and quantile functions to track percentile latency SLAs across your entire device fleet.
