# How to Migrate from Apache Druid to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Druid, Migration, Real-Time Analytics, Time Series, OLAP

Description: Migrate from Apache Druid to ClickHouse to simplify your operations, reduce infrastructure complexity, and maintain fast real-time analytical queries.

---

Apache Druid is a real-time analytics database with a complex architecture that includes Coordinator, Broker, Historical, MiddleManager, and Overlord nodes. ClickHouse delivers comparable query performance with a much simpler operational model.

## Architecture Comparison

| Aspect | Druid | ClickHouse |
|--------|-------|------------|
| Components | 6+ node types | Single binary (or replicated) |
| Ingestion | Kafka Indexing Service, Batch | Kafka Engine, INSERT, S3 |
| Storage | Deep storage (S3) + local cache | Local disk + S3 backup |
| Data model | Flat, rollup-optimized | Flexible, full row storage |
| Approximate functions | Theta sketches, HLL | uniq(), quantilesTDigest() |

## Step 1 - Export Druid Data

Use Druid's SQL API to export segments to CSV or JSON:

```bash
curl -X POST https://druid-broker:8082/druid/v2/sql \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "SELECT * FROM \"page_views\" WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '\''30'\'' DAY",
    "resultFormat": "csv",
    "header": true
  }' > page_views_export.csv
```

## Step 2 - Create the ClickHouse Table

Druid stores events with a `__time` dimension. Map this to a ClickHouse DateTime:

```sql
CREATE TABLE analytics.page_views
(
    event_time   DateTime,
    page         LowCardinality(String),
    user_id      String,
    country      LowCardinality(String),
    duration_ms  UInt32,
    views        UInt32 DEFAULT 1
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, page, user_id)
SETTINGS index_granularity = 8192;
```

## Step 3 - Load the Exported Data

```sql
INSERT INTO analytics.page_views
SELECT
    parseDateTimeBestEffort(`__time`) AS event_time,
    page,
    user_id,
    country,
    duration_ms,
    1 AS views
FROM file('/tmp/page_views_export.csv', 'CSV');
```

Or directly via S3:

```sql
INSERT INTO analytics.page_views
SELECT *
FROM s3('s3://bucket/druid-exports/*.csv', 'KEY', 'SECRET', 'CSV');
```

## Step 4 - Translate Druid SQL to ClickHouse SQL

Druid uses `TIME_FLOOR` for time bucketing:

```sql
-- Druid
SELECT TIME_FLOOR(__time, 'PT1H') AS hour, SUM(views) AS total
FROM page_views
GROUP BY 1
ORDER BY 1
```

ClickHouse equivalent:

```sql
SELECT toStartOfHour(event_time) AS hour, sum(views) AS total
FROM analytics.page_views
GROUP BY hour
ORDER BY hour
```

Druid approximate distinct count:

```sql
-- Druid
SELECT APPROX_COUNT_DISTINCT(user_id) FROM page_views
```

ClickHouse:

```sql
SELECT uniq(user_id) FROM analytics.page_views
```

## Step 5 - Set Up Real-Time Ingestion

If you were using Druid's Kafka Indexing Service, replace it with ClickHouse's Kafka engine:

```sql
CREATE TABLE analytics.page_views_kafka
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'page-views',
    kafka_group_name = 'clickhouse-consumer',
    kafka_format = 'JSONEachRow';

CREATE MATERIALIZED VIEW analytics.page_views_mv TO analytics.page_views AS
SELECT * FROM analytics.page_views_kafka;
```

## Summary

Migrating from Druid to ClickHouse simplifies your operational stack significantly. Data export, schema translation, and query rewriting are the main tasks. ClickHouse's time-based functions and approximate aggregation functions cover all common Druid query patterns, and its Kafka engine replaces Druid's streaming ingestion with minimal configuration.
