# ClickHouse vs Apache Doris for Real-Time Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Doris, Analytics, OLAP, Comparison, Real-Time

Description: Compare ClickHouse and Apache Doris for real-time analytics workloads, covering performance, architecture, SQL compatibility, and operational complexity.

---

Both ClickHouse and Apache Doris are open-source OLAP databases designed for real-time analytics. Choosing between them requires understanding their architectural trade-offs and where each excels.

## Architecture Overview

**ClickHouse** uses a shared-nothing architecture with MergeTree storage. Each node stores a shard of data independently. It is known for single-node raw speed, excellent compression, and a rich set of aggregate functions.

**Apache Doris** (formerly Apache Incubator Doris) uses a Frontend/Backend split. Frontend nodes handle query planning and metadata, while Backend nodes handle storage and execution. Doris prioritizes MySQL protocol compatibility and is optimized for ad-hoc multi-table joins.

## Performance Comparison

| Workload | ClickHouse | Apache Doris |
|---|---|---|
| Single-table aggregation | Excellent | Good |
| Multi-table JOIN | Good | Excellent |
| Real-time ingestion | Excellent | Good |
| Point lookups | Fast (with proper keys) | Fast |
| Full-text search | Limited | Limited |

ClickHouse consistently wins on single-table aggregation benchmarks. Doris tends to perform better on complex multi-table joins due to its cost-based optimizer.

## SQL Compatibility

Doris uses a MySQL-compatible dialect, which means:

```sql
-- Doris supports MySQL-style functions
SELECT DATE_FORMAT(created_at, '%Y-%m') AS month, COUNT(*) AS cnt
FROM events GROUP BY month;
```

ClickHouse uses its own dialect:

```sql
SELECT formatDateTime(created_at, '%Y-%m') AS month, count() AS cnt
FROM events GROUP BY month;
```

Doris's MySQL compatibility is a major advantage for teams already using MySQL tooling.

## Ingestion Model

ClickHouse ingestion via HTTP or Kafka:

```sql
INSERT INTO events
SELECT * FROM kafka_queue_table;
```

Doris ingestion via Stream Load:

```bash
curl -X PUT \
  -H "label: load_events_$(date +%s)" \
  -H "column_separator:," \
  -T events.csv \
  http://doris-fe:8030/api/mydb/events/_stream_load
```

## Storage Engines

ClickHouse offers multiple MergeTree variants for different workloads. Doris provides a similar set:

| ClickHouse | Apache Doris |
|---|---|
| MergeTree | Duplicate Key model |
| ReplacingMergeTree | Unique Key model |
| AggregatingMergeTree | Aggregate Key model |

## Operational Complexity

ClickHouse is generally simpler to operate on a single node or small cluster. Doris requires managing Frontend and Backend nodes separately, which adds complexity but improves query planning for complex workloads.

## When to Choose ClickHouse

- Single-table high-speed analytics
- Log analytics and observability
- Time-series metrics
- Teams comfortable with ClickHouse SQL dialect

## When to Choose Apache Doris

- Heavy multi-table JOIN workloads
- Teams migrating from MySQL needing protocol compatibility
- Mixed OLAP + partial OLTP workloads

## Summary

ClickHouse outperforms Apache Doris on single-table aggregation and raw throughput, while Doris excels at complex joins and MySQL compatibility. For most analytics-first workloads with moderate join complexity, ClickHouse is the faster and simpler choice.
