# How to Build Cloud Resource Utilization Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cloud Resource, Analytics, Utilization, Monitoring

Description: Learn how to build scalable cloud resource utilization analytics using ClickHouse to track CPU, memory, and cost data across your infrastructure.

---

## Why Cloud Resource Analytics Matter

Cloud infrastructure costs can spiral quickly without visibility. ClickHouse is well-suited for storing and querying time-series utilization data at scale, letting you slice by region, account, and service.

## Schema Design

```sql
CREATE TABLE cloud_resource_utilization (
    event_time DateTime,
    resource_id String,
    resource_type LowCardinality(String),
    region LowCardinality(String),
    account_id String,
    cpu_percent Float32,
    memory_percent Float32,
    disk_read_bytes UInt64,
    disk_write_bytes UInt64,
    network_in_bytes UInt64,
    network_out_bytes UInt64,
    cost_usd Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (resource_type, region, resource_id, event_time);
```

## Ingesting Data

Use the HTTP interface or a Kafka topic to ingest metrics. A typical row insert looks like:

```sql
INSERT INTO cloud_resource_utilization VALUES (
    now(), 'i-0abc123', 'ec2', 'us-east-1', '123456789',
    45.2, 72.1, 10485760, 5242880, 20971520, 10485760, 0.023
);
```

## Top Utilization Queries

Find the most CPU-intensive resources over the last 24 hours:

```sql
SELECT
    resource_id,
    resource_type,
    region,
    avg(cpu_percent) AS avg_cpu,
    max(cpu_percent) AS peak_cpu
FROM cloud_resource_utilization
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY resource_id, resource_type, region
ORDER BY avg_cpu DESC
LIMIT 20;
```

Identify underutilized resources for rightsizing:

```sql
SELECT
    resource_id,
    region,
    avg(cpu_percent) AS avg_cpu,
    avg(memory_percent) AS avg_mem,
    sum(cost_usd) AS total_cost
FROM cloud_resource_utilization
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY resource_id, region
HAVING avg_cpu < 10 AND avg_mem < 20
ORDER BY total_cost DESC;
```

## Cost Attribution

Aggregate daily spend per account and resource type:

```sql
SELECT
    toDate(event_time) AS day,
    account_id,
    resource_type,
    sum(cost_usd) AS daily_cost
FROM cloud_resource_utilization
GROUP BY day, account_id, resource_type
ORDER BY day DESC, daily_cost DESC;
```

## Using Materialized Views for Aggregates

Pre-aggregate hourly stats to speed up dashboards:

```sql
CREATE MATERIALIZED VIEW cloud_resource_hourly
ENGINE = SummingMergeTree()
ORDER BY (resource_type, region, resource_id, hour)
AS
SELECT
    toStartOfHour(event_time) AS hour,
    resource_id,
    resource_type,
    region,
    avg(cpu_percent) AS avg_cpu,
    avg(memory_percent) AS avg_mem,
    sum(cost_usd) AS total_cost
FROM cloud_resource_utilization
GROUP BY hour, resource_id, resource_type, region;
```

## Summary

ClickHouse is an excellent fit for cloud resource utilization analytics. With a partitioned MergeTree table, you can efficiently query utilization trends, spot cost anomalies, and identify idle resources. Combine raw data tables with materialized views to serve both ad-hoc queries and fast dashboard aggregations.
