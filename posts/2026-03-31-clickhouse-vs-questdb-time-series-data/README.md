# ClickHouse vs QuestDB for Time-Series Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, QuestDB, Time-Series, Metrics, TSDB, Analytics, Ingestion

Description: Compare ClickHouse and QuestDB for time-series data storage - examining ingestion throughput, query speed, SQL support, and production readiness.

---

Both ClickHouse and QuestDB are used for time-series workloads like metrics storage, IoT data, and financial tick data. They share columnar storage and fast aggregations but differ in their design tradeoffs. This post compares them to help you choose the right database for your time-series use case.

## Design Philosophy

QuestDB is purpose-built for time-series data. Its storage model is optimized around time-partitioned, append-only tables with out-of-order ingestion support via the ILP (InfluxDB Line Protocol). ClickHouse is a general-purpose OLAP database that handles time-series very well due to its columnar storage and efficient compression codecs for monotonically increasing data.

```text
QuestDB:    purpose-built TSDB | ILP protocol | out-of-order ingestion
ClickHouse: general OLAP + TSDB | native ingestion | DoubleDelta/Gorilla codecs
```

## Ingestion Performance

QuestDB claims extremely high ingestion rates via ILP, often exceeding 1 million rows/second on a single node. ClickHouse is also fast but performs best with batched inserts:

```sql
-- ClickHouse: optimized time-series table
CREATE TABLE metrics (
    ts       DateTime64(3) CODEC(DoubleDelta, ZSTD(1)),
    host     LowCardinality(String),
    metric   String,
    value    Float64 CODEC(Gorilla, ZSTD(1))
) ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (host, metric, ts)
TTL ts + INTERVAL 90 DAY;
```

## Query Performance

ClickHouse excels at complex aggregations over large time ranges:

```sql
SELECT
    toStartOfMinute(ts)        AS minute,
    host,
    avg(value)                 AS avg_val,
    max(value)                 AS max_val,
    quantile(0.99)(value)      AS p99
FROM metrics
WHERE ts >= now() - INTERVAL 24 HOUR
  AND metric = 'cpu_usage'
GROUP BY minute, host
ORDER BY minute;
```

QuestDB has a SAMPLE BY clause designed specifically for time-series downsampling:

```sql
-- QuestDB: native time-series aggregation
SELECT ts, avg(value), max(value)
FROM metrics
WHERE ts >= dateadd('d', -1, now())
SAMPLE BY 1m;
```

## SQL Compatibility

ClickHouse has richer SQL support - window functions, joins, subqueries, CTEs, and a vast library of built-in functions. QuestDB's SQL is PostgreSQL-compatible with time-series extensions but covers fewer advanced analytical patterns.

## Ecosystem and Integrations

ClickHouse integrates with Kafka, Grafana, dbt, Airflow, and dozens of other tools. QuestDB integrates with Grafana, supports the PostgreSQL wire protocol, and accepts InfluxDB Line Protocol - making it a drop-in replacement for InfluxDB in many setups.

## Production Maturity

ClickHouse has a larger production install base, more active community, and better enterprise support options. QuestDB is younger but has grown rapidly and is production-ready for focused time-series use cases.

## When to Choose ClickHouse

- Mixed workloads: time-series plus event analytics, logs, or traces
- Complex analytical queries beyond time-series aggregations
- Existing Kafka or ClickHouse ecosystem investment
- Large-scale deployments needing distributed operation

## When to Choose QuestDB

- Pure time-series workloads (IoT, metrics, financial data)
- Existing InfluxDB deployments being migrated
- Simpler operational setup for single-node time-series
- PostgreSQL wire protocol compatibility required

## Summary

ClickHouse is the more versatile choice - it handles time-series well and extends to all analytical use cases. QuestDB is a compelling option for teams focused exclusively on time-series data who want a purpose-built solution with native ILP support. For observability platforms combining metrics, logs, and traces, ClickHouse's versatility makes it the better foundation.
