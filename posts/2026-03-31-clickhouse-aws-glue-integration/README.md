# How to Integrate ClickHouse with AWS Glue

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AWS Glue, ETL, S3, Data Catalog, AWS

Description: Integrate ClickHouse with AWS Glue to run managed ETL jobs that load and transform data between S3, Glue Data Catalog, and ClickHouse.

---

## Overview

AWS Glue is a serverless ETL service that can read from and write to a wide range of data stores. Integrating ClickHouse with Glue lets you schedule managed ETL jobs, use Glue Crawlers to catalog ClickHouse-exported Parquet files, and build fully managed data pipelines without managing servers.

## Integration Patterns

There are two main ways to integrate:

```text
1. Glue Job -> ClickHouse JDBC driver (read/write ClickHouse directly)
2. ClickHouse -> S3 (export) -> Glue Crawler -> Athena/Redshift
```

## Pattern 1: Glue Job with ClickHouse JDBC

Upload the ClickHouse JDBC driver to S3, then reference it in your Glue job:

```bash
aws s3 cp clickhouse-jdbc-0.6.0-all.jar s3://my-bucket/jars/
```

In your Glue Python Shell or Spark job:

```python
import jaydebeapi

conn = jaydebeapi.connect(
    'com.clickhouse.jdbc.ClickHouseDriver',
    'jdbc:clickhouse://clickhouse-host:8123/analytics',
    ['default', 'password'],
    '/tmp/clickhouse-jdbc.jar'
)

cursor = conn.cursor()
cursor.execute("SELECT count() FROM events WHERE event_ts >= today() - 7")
rows = cursor.fetchall()
print(rows)
```

## Pattern 2: Export to S3 and Catalog with Glue

Export ClickHouse data to S3 in Parquet format:

```sql
INSERT INTO FUNCTION s3(
    's3://my-bucket/exports/events/year=2026/month=03/data.parquet',
    'KEY', 'SECRET', 'Parquet'
)
SELECT * FROM events
WHERE toYYYYMM(event_ts) = 202603;
```

Create a Glue Crawler to catalog the exported files:

```bash
aws glue create-crawler \
  --name clickhouse-events-crawler \
  --role GlueRole \
  --database-name analytics \
  --targets '{"S3Targets": [{"Path": "s3://my-bucket/exports/events/"}]}'
```

Run the crawler:

```bash
aws glue start-crawler --name clickhouse-events-crawler
```

Now query the data via Athena:

```sql
SELECT event_type, count(*)
FROM analytics.events
WHERE year = 2026 AND month = 3
GROUP BY event_type;
```

## Load Glue-Processed Data into ClickHouse

After a Glue ETL job transforms data and writes it back to S3, import it into ClickHouse:

```sql
INSERT INTO events_processed
SELECT *
FROM s3(
    's3://my-bucket/processed/events/*.parquet',
    'KEY', 'SECRET', 'Parquet'
);
```

## Schedule with Glue Triggers

Automate the pipeline with a Glue scheduled trigger:

```bash
aws glue create-trigger \
  --name daily-clickhouse-export \
  --type SCHEDULED \
  --schedule 'cron(0 2 * * ? *)' \
  --actions '[{"JobName": "clickhouse-export-job"}]'
```

## Summary

ClickHouse integrates with AWS Glue either via JDBC for direct read/write access or through S3 as an intermediate store. The S3-based approach is recommended for large-scale ETL because it decouples the compute (Glue) from the storage (ClickHouse), allows parallel reads, and fits naturally into the Glue Data Catalog ecosystem.
