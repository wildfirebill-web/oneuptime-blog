# ClickHouse for Redshift Users - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Redshift, AWS, Data Warehouse, Migration

Description: A guide for Amazon Redshift users comparing ClickHouse on SQL compatibility, cluster management, performance, and cost efficiency.

---

Amazon Redshift and ClickHouse are both columnar databases designed for analytical workloads. Redshift is AWS's managed data warehouse based on PostgreSQL. ClickHouse is an open-source database that typically delivers faster query performance at lower cost, especially for high-cardinality aggregate queries.

## SQL Compatibility

Redshift SQL is PostgreSQL-based. ClickHouse has its own dialect with differences in function names and syntax:

```sql
-- Redshift: PostgreSQL-style date/time
SELECT DATE_TRUNC('hour', event_time) AS hour
FROM events;

SELECT DATEDIFF('day', start_date, GETDATE()) AS days_ago;

-- ClickHouse equivalents
SELECT toStartOfHour(event_time) AS hour
FROM events;

SELECT dateDiff('day', start_date, today()) AS days_ago;
```

## Distribution Keys vs. Sorting Keys

Redshift distributes data across nodes using a distribution key. ClickHouse distributes via sharding and sorts via `ORDER BY`:

```sql
-- Redshift: distribution and sort keys
CREATE TABLE events (
    event_time TIMESTAMP,
    user_id INTEGER,
    event_type VARCHAR(50)
)
DISTKEY(user_id)
SORTKEY(event_time);

-- ClickHouse equivalent
CREATE TABLE events (
    event_time DateTime,
    user_id UInt32,
    event_type LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

## VACUUM and ANALYZE vs. Automatic Merges

Redshift requires periodic `VACUUM` to reclaim space from deleted rows and `ANALYZE` to update statistics. ClickHouse handles merges automatically in the background via the MergeTree engine.

```sql
-- Redshift: manual maintenance
VACUUM FULL events;
ANALYZE events;

-- ClickHouse: merges happen automatically
-- You can trigger manually if needed
OPTIMIZE TABLE events FINAL;
```

## COPY Command vs. INSERT

Redshift uses the `COPY` command to load data from S3 in bulk. ClickHouse can read directly from S3 using the `s3` table function:

```sql
-- Redshift: COPY from S3
COPY events FROM 's3://mybucket/events/'
IAM_ROLE 'arn:aws:iam::123:role/RedshiftRole'
FORMAT AS PARQUET;

-- ClickHouse: direct S3 query or insert
INSERT INTO events
SELECT * FROM s3(
    's3://mybucket/events/*.parquet',
    'access_key', 'secret_key',
    'Parquet'
);
```

## Concurrency Scaling

Redshift has a fixed number of concurrency slots based on cluster size. Adding concurrent queries beyond the slot count causes queuing. ClickHouse handles high concurrency more gracefully due to its internal query scheduling.

## Cluster Resizing

Redshift classic resize takes hours and makes the cluster read-only. ClickHouse scaling is done by adding shards or replicas with no downtime.

## Summary

Redshift users moving to ClickHouse will find better performance for high-cardinality aggregations, lower storage costs due to better compression, and no need for manual VACUUM operations. The primary trade-off is moving from AWS's fully managed experience to either self-hosting or ClickHouse Cloud.
