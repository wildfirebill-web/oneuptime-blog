# How to Use ClickHouse as a Prometheus Long-Term Storage Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Prometheus, Long-Term Storage, Observability, Metric

Description: Learn how to configure ClickHouse as a remote write storage backend for Prometheus to retain metrics for months or years at low cost.

---

## Why ClickHouse for Prometheus Long-Term Storage

Prometheus's local TSDB retains data for a configurable window (default 15 days). ClickHouse can receive Prometheus remote write data, compress it efficiently, and serve range queries through a compatible adapter - providing cheap multi-year retention.

## Adapter Options

The most popular adapters are:

- **Promhouse** - early open-source adapter
- **clickhouse_exporter with remote write** - Percona's stack
- **Promxy + ClickHouse** - federation-aware
- **Grafana Metricstank** (partial support)
- **DoubleCloud Prometheus-ClickHouse adapter** (recommended for new deployments)

## Installing the DoubleCloud Adapter

```bash
docker run -d \
  --name prometheus-clickhouse \
  -p 9201:9201 \
  -e CLICKHOUSE_URL=http://clickhouse:8123 \
  -e CLICKHOUSE_DATABASE=metrics \
  doublecloud/prometheus-remote-storage-clickhouse:latest
```

## ClickHouse Schema

The adapter creates a table similar to:

```sql
CREATE TABLE metrics.samples (
  date        Date DEFAULT toDate(timestamp),
  name        LowCardinality(String),
  tags        Array(String),
  val         Float64,
  timestamp   DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (name, tags, timestamp)
TTL date + INTERVAL 1 YEAR
```

## Prometheus remote_write Configuration

```yaml
remote_write:
  - url: http://prometheus-clickhouse:9201/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 100000
      max_shards: 10

remote_read:
  - url: http://prometheus-clickhouse:9201/read
    read_recent: true
```

## Querying Long-Term Data in Grafana

Point a Grafana Prometheus data source at the adapter's `/read` endpoint. Long-range queries work transparently.

```text
Datasource URL: http://prometheus-clickhouse:9201
```

For direct ClickHouse queries, add a ClickHouse Grafana plugin data source to build custom dashboards:

```sql
SELECT
  toStartOfHour(timestamp) AS time,
  avg(val) AS cpu_avg
FROM metrics.samples
WHERE name = 'node_cpu_seconds_total'
  AND timestamp BETWEEN {from:DateTime} AND {to:DateTime}
GROUP BY time
ORDER BY time
```

## Retention Tuning

Adjust the TTL clause to match your retention policy:

```sql
ALTER TABLE metrics.samples MODIFY TTL date + INTERVAL 2 YEAR
```

## Summary

ClickHouse stores Prometheus metrics via a remote write adapter, providing compressed multi-year retention at a fraction of the cost of dedicated TSDB solutions. Use an adapter like DoubleCloud's Prometheus-ClickHouse bridge, configure Prometheus `remote_write` and `remote_read` endpoints, and leverage ClickHouse's native TTL for automated data expiry.
