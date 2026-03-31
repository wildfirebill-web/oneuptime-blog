# How to Migrate from Apache Hive to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Hive, Migration, HDFS, Analytics, OLAP, Hadoop

Description: Migrate analytical workloads from Apache Hive to ClickHouse by reading Hive data directly via the Hive engine or exporting through Parquet on HDFS or S3.

---

Apache Hive delivers batch SQL on Hadoop, but query latencies in the minutes or hours range do not suit interactive analytics. ClickHouse answers the same queries in seconds. This guide covers two migration paths: the ClickHouse Hive engine for live access, and Parquet export for a full migration.

## Option 1 - ClickHouse Hive Table Engine

ClickHouse can query Hive tables directly via the Hive engine, without copying data:

```sql
CREATE TABLE hive_events
ENGINE = Hive(
    'thrift://hive-metastore:9083',
    'analytics',
    'events'
)
PARTITION BY (year, month);
```

Query it to verify access:

```sql
SELECT event_type, count() FROM hive_events
WHERE year = 2024 AND month = 1
GROUP BY event_type
LIMIT 10;
```

Then copy to a local MergeTree table:

```sql
INSERT INTO events
SELECT * FROM hive_events
WHERE year = 2024;
```

## Option 2 - Export Hive Data as Parquet

In HiveQL, write to Parquet on HDFS:

```sql
INSERT OVERWRITE DIRECTORY '/tmp/events_export'
STORED AS PARQUET
SELECT * FROM events;
```

Or write directly to S3 if using S3-compatible storage:

```sql
INSERT OVERWRITE DIRECTORY 's3a://my-bucket/hive-export/events/'
STORED AS PARQUET
SELECT * FROM events;
```

## Load Parquet into ClickHouse

From HDFS:

```bash
hdfs dfs -copyToLocal /tmp/events_export /tmp/events_local/

clickhouse-client \
  --query "INSERT INTO events FORMAT Parquet" \
  < /tmp/events_local/000000_0
```

From S3:

```sql
INSERT INTO events
SELECT *
FROM s3(
    'https://s3.amazonaws.com/my-bucket/hive-export/events/*.parquet',
    'ACCESS_KEY',
    'SECRET_KEY',
    'Parquet'
);
```

## Translating the Schema

Hive DDL:

```sql
CREATE TABLE events (
    event_id   BIGINT,
    user_id    BIGINT,
    event_type STRING,
    created_at TIMESTAMP,
    revenue    DOUBLE
)
PARTITIONED BY (year INT, month INT)
STORED AS ORC;
```

ClickHouse equivalent (dropping Hive partition columns, using native partitioning):

```sql
CREATE TABLE events (
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    revenue    Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## HiveQL to ClickHouse Query Translation

HiveQL:

```sql
SELECT
    event_type,
    COUNT(*) AS cnt,
    SUM(revenue) AS total
FROM events
WHERE year = 2024 AND month BETWEEN 1 AND 6
GROUP BY event_type
ORDER BY total DESC
LIMIT 10;
```

ClickHouse equivalent:

```sql
SELECT
    event_type,
    count() AS cnt,
    sum(revenue) AS total
FROM events
WHERE toYYYYMM(created_at) BETWEEN 202401 AND 202406
GROUP BY event_type
ORDER BY total DESC
LIMIT 10;
```

## Summary

Migrating from Apache Hive to ClickHouse reduces query latency from minutes to seconds. The Hive table engine allows a live migration approach - run ClickHouse against existing Hive data while gradually moving tables, then switch workloads over once validated.
