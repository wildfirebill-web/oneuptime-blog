# How to Build Container Monitoring with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Container Monitoring, Docker, cAdvisor, Resource Utilization

Description: Learn how to ingest container resource metrics from cAdvisor into ClickHouse and build dashboards for CPU, memory, and network utilization.

---

Container monitoring requires tracking CPU throttling, memory pressure, network I/O, and disk usage at per-container granularity. ClickHouse stores these high-frequency metric samples efficiently and answers aggregation queries in milliseconds, making it a solid backend for container observability platforms.

## Schema

```sql
CREATE TABLE container_metrics
(
    ts                DateTime,
    host              LowCardinality(String),
    container_id      FixedString(12),
    container_name    LowCardinality(String),
    image             LowCardinality(String),
    cpu_usage_ns      UInt64,  -- nanoseconds
    cpu_throttled_ns  UInt64,
    mem_usage_bytes   UInt64,
    mem_limit_bytes   UInt64,
    net_rx_bytes      UInt64,
    net_tx_bytes      UInt64,
    blk_read_bytes    UInt64,
    blk_write_bytes   UInt64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (host, container_name, ts);
```

## CPU Utilization Percent

cAdvisor gives cumulative nanoseconds, so differentiate between samples:

```sql
SELECT
    container_name,
    toStartOfMinute(ts) AS minute,
    (max(cpu_usage_ns) - min(cpu_usage_ns)) / 1e9 /
        dateDiff('second', min(ts), max(ts)) * 100 AS cpu_pct
FROM container_metrics
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY container_name, minute
ORDER BY container_name, minute;
```

## Memory Utilization Percent

```sql
SELECT
    container_name,
    toStartOfMinute(ts)                                AS minute,
    avg(mem_usage_bytes) * 100.0 / max(mem_limit_bytes) AS mem_pct
FROM container_metrics
WHERE ts >= now() - INTERVAL 1 HOUR
  AND mem_limit_bytes > 0
GROUP BY container_name, minute
ORDER BY container_name, minute;
```

## CPU Throttling Rate

```sql
SELECT
    container_name,
    toStartOfHour(ts) AS hour,
    sum(cpu_throttled_ns) * 100.0 / (sum(cpu_usage_ns) + 1) AS throttle_rate_pct
FROM container_metrics
WHERE ts >= now() - INTERVAL 24 HOUR
GROUP BY container_name, hour
ORDER BY throttle_rate_pct DESC;
```

## Network I/O by Container

```sql
SELECT
    container_name,
    formatReadableSize(sum(net_rx_bytes)) AS rx,
    formatReadableSize(sum(net_tx_bytes)) AS tx
FROM container_metrics
WHERE ts >= toStartOfDay(now())
GROUP BY container_name
ORDER BY sum(net_rx_bytes) + sum(net_tx_bytes) DESC
LIMIT 20;
```

## Top Memory-Consuming Containers

```sql
SELECT
    container_name,
    formatReadableSize(avg(mem_usage_bytes)) AS avg_mem,
    formatReadableSize(max(mem_usage_bytes)) AS peak_mem
FROM container_metrics
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY container_name
ORDER BY avg(mem_usage_bytes) DESC
LIMIT 20;
```

## Block I/O Trend

```sql
SELECT
    container_name,
    toStartOfMinute(ts) AS minute,
    sum(blk_read_bytes) / 1048576  AS mb_read,
    sum(blk_write_bytes) / 1048576 AS mb_written
FROM container_metrics
WHERE container_name = 'postgres'
  AND ts >= now() - INTERVAL 2 HOUR
GROUP BY container_name, minute
ORDER BY minute;
```

## Summary

ClickHouse is an efficient long-term store for container metrics from cAdvisor or similar collectors. Store per-sample data in a partitioned MergeTree table, then compute CPU utilization from cumulative counters, track memory pressure against limits, and identify throttled or network-heavy containers. This approach supports months of retention at far lower cost than purpose-built TSDB solutions.
