# How to Use s3Cluster() Table Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, s3Cluster(), Table Function, S3, Distributed, Analytics, AWS

Description: Learn how to use the s3Cluster() table function to parallelize S3 file reads across all nodes in a ClickHouse cluster for high-throughput data lake queries.

---

`s3Cluster(cluster_name, url, ...)` is the distributed counterpart of the `s3()` table function. Instead of reading S3 files on the initiating node only, `s3Cluster()` distributes the file list across all nodes in the named cluster and parallelizes the read. This dramatically reduces query time when reading large numbers of S3 files.

## Syntax

```sql
s3Cluster(
    'cluster_name',
    's3_url_or_glob',
    ['access_key', 'secret_key'],
    'format',
    ['structure'],
    ['compression_method']
)
```

## Basic Example: Distributed S3 Read

```sql
-- Read all Parquet files in a prefix across the cluster
SELECT count()
FROM s3Cluster(
    'analytics_cluster',
    's3://my-data-lake/events/2026/*.parquet',
    'access_key_id',
    'secret_access_key',
    'Parquet'
);
```

## Aggregating S3 Data Across Nodes

```sql
SELECT
    event_type,
    count() AS cnt,
    uniqExact(user_id) AS unique_users
FROM s3Cluster(
    'analytics_cluster',
    's3://my-data-lake/events/2026/03/*.parquet',
    'access_key_id',
    'secret_access_key',
    'Parquet'
)
WHERE toDate(event_time) BETWEEN '2026-03-01' AND '2026-03-31'
GROUP BY event_type
ORDER BY cnt DESC;
```

## Comparing s3() vs s3Cluster()

```text
Function       Reads on         Best For
s3()           Single node      Small files, single-node ClickHouse
s3Cluster()    All cluster nodes  Large file sets, production clusters
```

For a 10-node cluster reading 1000 files, `s3Cluster()` assigns roughly 100 files per node, giving near-linear speedup.

## Loading S3 Data into ClickHouse

```sql
-- Distributed ingest from S3 into a local table
INSERT INTO events (event_time, event_type, user_id, amount)
SELECT event_time, event_type, user_id, amount
FROM s3Cluster(
    'analytics_cluster',
    's3://my-data-lake/events/2026/01/*.parquet',
    'access_key_id',
    'secret_access_key',
    'Parquet'
);
```

## Using IAM Role Instead of Keys

On AWS EC2 with an IAM instance profile, omit the credentials:

```sql
SELECT count()
FROM s3Cluster(
    'analytics_cluster',
    's3://my-data-lake/events/2026/03/*.parquet',
    'Parquet'
);
```

ClickHouse picks up the instance profile credentials automatically.

## Specifying Structure Explicitly

If schema inference fails or you want to control types:

```sql
SELECT user_id, sum(amount) AS total
FROM s3Cluster(
    'analytics_cluster',
    's3://my-data-lake/orders/*.csv',
    'CSVWithNames',
    'user_id UInt64, amount Float64, order_date Date'
)
GROUP BY user_id
ORDER BY total DESC
LIMIT 100;
```

## Monitoring Distributed S3 Queries

```sql
SELECT
    query_id,
    query,
    elapsed,
    read_bytes,
    read_rows
FROM system.processes
WHERE query LIKE '%s3Cluster%';
```

## Cluster Configuration

Ensure the cluster is defined in `remote_servers` and all nodes have network access to S3. Each node reads its assigned file subset independently, then results are aggregated on the initiating node.

## Summary

`s3Cluster()` parallelizes S3 reads across all nodes in a ClickHouse cluster, making it the right choice for large data lake queries. It accepts the same parameters as `s3()` but adds a cluster name as the first argument. Use it for high-throughput analytical queries and bulk S3-to-ClickHouse ingestion on multi-node deployments.
