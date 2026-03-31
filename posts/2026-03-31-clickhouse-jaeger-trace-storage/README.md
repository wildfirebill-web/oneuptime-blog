# How to Use ClickHouse with Jaeger for Trace Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Jaeger, Distributed Tracing, Observability, OpenTelemetry

Description: Learn how to configure Jaeger to use ClickHouse as its trace storage backend for scalable, long-term distributed trace retention.

---

## Why ClickHouse as Jaeger Storage

Jaeger's default storage backends (Cassandra, Elasticsearch) are operationally heavy and expensive at scale. ClickHouse offers high ingest throughput, excellent query performance for span searches, and cheap long-term retention - making it a strong backend for production Jaeger deployments.

## jaeger-clickhouse Plugin

The `jaeger-clickhouse` storage plugin wraps ClickHouse behind Jaeger's storage gRPC plugin API.

```bash
docker pull ghcr.io/jaegertracing/jaeger-clickhouse:latest
```

## ClickHouse Schema

The plugin creates these tables automatically on startup:

```sql
CREATE TABLE jaeger_spans (
  timestamp     DateTime64(9) CODEC(Delta, ZSTD(1)),
  trace_id      FixedString(16),
  span_id       UInt64,
  parent_span_id UInt64,
  operation_name LowCardinality(String),
  service_name  LowCardinality(String),
  duration_us   Int64,
  tags          Array(Tuple(String, String)),
  process_tags  Array(Tuple(String, String)),
  logs          Array(Tuple(DateTime64(9), Array(Tuple(String, String))))
) ENGINE = MergeTree()
PARTITION BY toDate(timestamp)
ORDER BY (service_name, -toUnixTimestamp(timestamp), trace_id)
TTL toDate(timestamp) + INTERVAL 30 DAY
```

## Docker Compose Example

```yaml
version: '3.8'
services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-data:/var/lib/clickhouse

  jaeger-clickhouse-plugin:
    image: ghcr.io/jaegertracing/jaeger-clickhouse:latest
    environment:
      CLICKHOUSE_URL: tcp://clickhouse:9000
      CLICKHOUSE_DATABASE: jaeger
    ports:
      - "17271:17271"  # gRPC plugin port

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      SPAN_STORAGE_TYPE: grpc-plugin
      GRPC_STORAGE_PLUGIN_BINARY: /jaeger-clickhouse
      GRPC_STORAGE_PLUGIN_CONFIGURATION_FILE: /config.yaml
    ports:
      - "16686:16686"
      - "14268:14268"
    depends_on:
      - jaeger-clickhouse-plugin

volumes:
  clickhouse-data:
```

## Querying Traces in ClickHouse Directly

```sql
SELECT
  trace_id,
  service_name,
  operation_name,
  duration_us / 1000 AS duration_ms
FROM jaeger_spans
WHERE
  service_name = 'payment-service'
  AND timestamp >= now() - INTERVAL 1 HOUR
  AND duration_us > 500000  -- slower than 500ms
ORDER BY duration_us DESC
LIMIT 20
```

## Retention Configuration

Adjust the TTL to match your retention policy:

```sql
ALTER TABLE jaeger_spans MODIFY TTL toDate(timestamp) + INTERVAL 90 DAY
```

## Summary

Use the `jaeger-clickhouse` gRPC storage plugin to send Jaeger spans to ClickHouse. The plugin auto-creates optimized tables with TTL expiry. ClickHouse's columnar storage makes span attribute filtering and trace reconstruction significantly faster and cheaper than Elasticsearch at high trace volumes. Query traces directly in ClickHouse for advanced analysis beyond what Jaeger's UI supports.
