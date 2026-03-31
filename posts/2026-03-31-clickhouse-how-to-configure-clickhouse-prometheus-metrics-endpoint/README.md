# How to Configure ClickHouse Prometheus Metrics Endpoint

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Prometheus, Monitoring, Metric, Observability

Description: Learn how to configure ClickHouse's built-in Prometheus metrics endpoint to expose server metrics for scraping by Prometheus and visualization in Grafana.

---

## Overview

ClickHouse has a built-in HTTP endpoint that exposes server metrics in Prometheus format. This enables standard Prometheus scraping without requiring external exporters. The endpoint is configured in `config.xml` or in a file under `config.d/`.

## Enabling the Prometheus Endpoint

Add or edit the `<prometheus>` section in your ClickHouse configuration:

```xml
<!-- /etc/clickhouse-server/config.d/prometheus.xml -->
<clickhouse>
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
        <status_info>true</status_info>
    </prometheus>
</clickhouse>
```

Restart ClickHouse after adding this configuration:

```bash
sudo systemctl restart clickhouse-server
```

## Verifying the Endpoint

```bash
curl http://localhost:9363/metrics | head -40
```

Expected output:

```text
# HELP ClickHouseMetrics_Query Number of executing queries
# TYPE ClickHouseMetrics_Query gauge
ClickHouseMetrics_Query 2
# HELP ClickHouseMetrics_Merge Number of executing background merges
# TYPE ClickHouseMetrics_Merge gauge
ClickHouseMetrics_Merge 1
# HELP ClickHouseEvents_Query Number of queries to be interpreted and potentially executed
# TYPE ClickHouseEvents_Query counter
ClickHouseEvents_Query 14523
```

## Metric Categories

The endpoint exposes three categories of metrics:

**Metrics** (`metrics: true`) - current gauge values:
- `ClickHouseMetrics_Query` - number of running queries
- `ClickHouseMetrics_Merge` - number of background merges
- `ClickHouseMetrics_Connection` - active connections
- `ClickHouseMetrics_MemoryTracking` - allocated memory

**Events** (`events: true`) - cumulative counters:
- `ClickHouseEvents_Query` - total queries executed
- `ClickHouseEvents_InsertedRows` - total rows inserted
- `ClickHouseEvents_ReadCompressedBytes` - bytes read

**Asynchronous Metrics** (`asynchronous_metrics: true`) - system-level metrics updated periodically:
- `ClickHouseAsyncMetrics_MemoryResident` - RSS memory
- `ClickHouseAsyncMetrics_Uptime` - server uptime
- `ClickHouseAsyncMetrics_MaxPartCountForPartition` - max parts per partition

## Configuring Prometheus Scrape

Add ClickHouse to your Prometheus `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'clickhouse'
    static_configs:
      - targets:
          - 'clickhouse-host-1:9363'
          - 'clickhouse-host-2:9363'
    metrics_path: /metrics
    scrape_interval: 15s
    scrape_timeout: 10s
```

## Securing the Endpoint

To restrict access, use ClickHouse's built-in HTTP handler authentication or place a reverse proxy (nginx/HAProxy) in front of port 9363:

```xml
<prometheus>
    <endpoint>/metrics</endpoint>
    <port>9363</port>
    <metrics>true</metrics>
    <events>true</events>
    <asynchronous_metrics>true</asynchronous_metrics>
    <!-- Bind to localhost only for security -->
    <address>127.0.0.1</address>
</prometheus>
```

## Key Metrics to Alert On

```yaml
# prometheus alert rules
groups:
  - name: clickhouse
    rules:
      - alert: ClickHouseTooManyParts
        expr: ClickHouseAsyncMetrics_MaxPartCountForPartition > 300
        for: 5m
        labels:
          severity: warning

      - alert: ClickHouseHighMemoryUsage
        expr: ClickHouseAsyncMetrics_MemoryResident > 50e9
        for: 2m
        labels:
          severity: critical

      - alert: ClickHouseQueryFailureRate
        expr: rate(ClickHouseEvents_FailedQuery[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
```

## Grafana Dashboard

Import the official ClickHouse Grafana dashboard (ID 14268) from grafana.com after configuring your Prometheus data source. It provides pre-built panels for query rates, memory usage, merge activity, and replication lag.

## Summary

ClickHouse's built-in Prometheus endpoint on port 9363 exposes metrics, events, and asynchronous system metrics without any external exporter. Enable it via the `<prometheus>` configuration section, scrape it with standard Prometheus configuration, and alert on key signals like part counts, memory usage, and query failure rates using the `ClickHouseMetrics_*`, `ClickHouseEvents_*`, and `ClickHouseAsyncMetrics_*` metric families.
