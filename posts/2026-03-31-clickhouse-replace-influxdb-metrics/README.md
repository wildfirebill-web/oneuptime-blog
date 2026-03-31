# How to Replace InfluxDB with ClickHouse for Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, InfluxDB, Metric, Time Series, Migration

Description: Replace InfluxDB with ClickHouse for metrics storage to gain SQL flexibility, better compression, and lower operational overhead at scale.

---

## Why Consider Replacing InfluxDB

InfluxDB is purpose-built for time series but comes with limitations at scale: the Flux query language has a steep learning curve, the storage engine struggles with high cardinality metrics, and clustering requires InfluxDB Enterprise (paid).

ClickHouse handles time-series metrics well with SQL, superior compression via codecs, and open-source clustering built in.

## Schema Design for Metrics

InfluxDB uses measurements, tags, and fields. Map these to ClickHouse columns:

```sql
CREATE TABLE metrics (
    metric_name  LowCardinality(String),
    timestamp    DateTime64(3) CODEC(Delta, ZSTD),
    -- tags (low cardinality dimensions)
    host         LowCardinality(String),
    region       LowCardinality(String),
    service      LowCardinality(String),
    -- fields (the actual measurements)
    value        Float64 CODEC(Gorilla, ZSTD),
    -- extra tags as a map for flexibility
    extra_tags   Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toDate(timestamp)
ORDER BY (metric_name, host, service, timestamp)
TTL toDate(timestamp) + INTERVAL 365 DAY DELETE;
```

The `Gorilla` codec is specifically designed for floating-point time-series data and matches InfluxDB's compression approach.

## Migrating Data from InfluxDB

Export data from InfluxDB using the CLI:

```bash
influx query 'from(bucket:"metrics") |> range(start: 2026-01-01, stop: 2026-04-01)' \
  --raw > metrics_export.csv
```

Transform and load into ClickHouse:

```bash
python3 influx_to_clickhouse.py metrics_export.csv | \
  clickhouse-client --query "INSERT INTO metrics FORMAT JSONEachRow"
```

## Updating Metrics Collectors

Replace the InfluxDB output in Telegraf with the ClickHouse output plugin:

```text
[[outputs.http]]
  url = "http://ch.internal:8123/?query=INSERT+INTO+metrics+FORMAT+JSONEachRow"
  method = "POST"
  data_format = "json"
  json_timestamp_units = "1ms"
```

For Prometheus remote write, use the ClickHouse Prometheus remote write adapter.

## Querying Metrics with SQL

Average CPU usage over 5-minute windows:

```sql
SELECT
    toStartOfFiveMinutes(timestamp) AS window,
    host,
    avg(value) AS avg_cpu
FROM metrics
WHERE metric_name = 'cpu_usage_percent'
  AND timestamp >= now() - INTERVAL 6 HOUR
GROUP BY window, host
ORDER BY window, host;
```

Compare this to the equivalent InfluxDB Flux query - the SQL version is more readable for most engineers.

## Handling High Cardinality

ClickHouse handles high cardinality well because tags are stored as columns, not indexed. For dynamic tags, use a `Map`:

```sql
SELECT extra_tags['kubernetes_pod'] AS pod, avg(value) AS avg_mem
FROM metrics
WHERE metric_name = 'memory_rss_bytes'
  AND timestamp >= now() - INTERVAL 1 HOUR
  AND extra_tags['namespace'] = 'production'
GROUP BY pod
ORDER BY avg_mem DESC
LIMIT 20;
```

## What InfluxDB Does Better

InfluxDB has native continuous queries and task scheduling that ClickHouse does not replicate out of the box. If your workflow relies heavily on InfluxDB tasks and flux transformations, factor in the migration effort.

## Summary

Replacing InfluxDB with ClickHouse for metrics storage provides SQL-based querying, better compression with time-series codecs, and simpler open-source clustering - making it a strong choice for teams already comfortable with SQL analytics.
