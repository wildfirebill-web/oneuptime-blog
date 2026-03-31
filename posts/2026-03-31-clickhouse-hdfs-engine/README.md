# How to Use HDFS Table Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, HDFS, Storage Engine, Hadoop, Data Lake

Description: Learn how to use the HDFS table engine in ClickHouse to read from and write to files on Hadoop HDFS, enabling integration with existing Hadoop data lakes.

---

The `HDFS` table engine allows ClickHouse to read from and write to files stored on Hadoop Distributed File System. It supports glob patterns for reading multiple files and works with common formats such as Parquet, ORC, CSV, JSON, and Avro. This engine is particularly useful when ClickHouse sits alongside an existing Hadoop ecosystem and you want to run fast analytical queries against HDFS data without copying it into ClickHouse's local storage.

## Prerequisites

ClickHouse must be able to reach the HDFS NameNode. Configure the HDFS connection in `config.xml` if your cluster uses Kerberos or non-default settings.

```xml
<!-- /etc/clickhouse-server/config.d/hdfs.xml -->
<yandex>
  <hdfs>
    <libhdfs3_conf>/etc/hadoop/conf/hdfs-site.xml</libhdfs3_conf>
  </hdfs>
</yandex>
```

## Creating an HDFS Table

```sql
-- HDFS engine syntax: HDFS('hdfs://namenode:port/path', 'Format')
CREATE TABLE hdfs_access_logs
(
    log_date    Date,
    host        String,
    request     String,
    status_code UInt16,
    bytes_sent  UInt64,
    referer     String,
    user_agent  String
)
ENGINE = HDFS(
    'hdfs://namenode:8020/data/access_logs/2024/06/*.csv',
    'CSV'
);
```

## Reading From HDFS

```sql
SELECT
    log_date,
    status_code,
    count()           AS request_count,
    sum(bytes_sent)   AS total_bytes
FROM hdfs_access_logs
WHERE log_date = '2024-06-15'
  AND status_code >= 400
GROUP BY log_date, status_code
ORDER BY request_count DESC;
```

```text
log_date    status_code  request_count  total_bytes
2024-06-15  404          8432           2109400
2024-06-15  500          127            63450
2024-06-15  403          44             10956
```

## Reading Parquet Files From HDFS

```sql
CREATE TABLE hdfs_events_parquet
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  String,
    page        String,
    duration_ms UInt32,
    country     String
)
ENGINE = HDFS(
    'hdfs://namenode:8020/warehouse/events/year=2024/month=06/*.parquet',
    'Parquet'
);

SELECT
    toDate(event_time) AS event_date,
    event_type,
    count()            AS events,
    uniq(user_id)      AS unique_users
FROM hdfs_events_parquet
WHERE event_time BETWEEN '2024-06-01' AND '2024-06-30'
GROUP BY event_date, event_type
ORDER BY event_date, events DESC;
```

## Writing Data to HDFS

```sql
-- Export ClickHouse data to HDFS as Parquet
INSERT INTO FUNCTION hdfs(
    'hdfs://namenode:8020/exports/daily_summary_20240615.parquet',
    'Parquet'
)
SELECT
    event_date,
    country,
    event_type,
    sum(event_count)  AS total_events,
    sum(revenue)      AS total_revenue
FROM daily_event_summary
WHERE event_date = yesterday()
GROUP BY event_date, country, event_type;
```

## Glob Patterns

```sql
-- Query all months in 2024
SELECT count()
FROM hdfs(
    'hdfs://namenode:8020/data/events/2024-{01..12}-*.parquet',
    'Parquet'
);

-- Query using wildcard
SELECT count()
FROM hdfs(
    'hdfs://namenode:8020/data/events/2024/*.csv',
    'CSV'
);
```

## Using the hdfs() Table Function

For ad hoc queries:

```sql
SELECT
    status_code,
    count() AS requests
FROM hdfs(
    'hdfs://namenode:8020/data/access_logs/2024/06/15.csv',
    'CSV'
)
WHERE status_code >= 500
GROUP BY status_code;
```

## Reading ORC Files

```sql
CREATE TABLE hdfs_orders_orc
(
    order_id     UInt64,
    customer_id  UInt64,
    status       String,
    total_amount Float64,
    created_at   DateTime
)
ENGINE = HDFS(
    'hdfs://namenode:8020/warehouse/orders/*.orc',
    'ORC'
);

SELECT
    status,
    count()            AS order_count,
    sum(total_amount)  AS total_revenue
FROM hdfs_orders_orc
WHERE toDate(created_at) = yesterday()
GROUP BY status
ORDER BY order_count DESC;
```

## Loading HDFS Data Into a Local MergeTree Table

For repeated querying, it is more efficient to copy HDFS data into ClickHouse local storage.

```sql
CREATE TABLE events_local
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    page        String,
    duration_ms UInt32,
    country     LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id);

-- One-time load from HDFS
INSERT INTO events_local
SELECT
    event_time,
    user_id,
    event_type,
    page,
    duration_ms,
    country
FROM hdfs_events_parquet
WHERE toYYYYMM(event_time) = 202406;
```

## Joining HDFS Files With Local Tables

```sql
-- Enrich HDFS event data with a local dimension table
SELECT
    e.event_type,
    u.plan,
    u.country,
    count()          AS events,
    uniq(e.user_id)  AS unique_users
FROM hdfs_events_parquet AS e
JOIN users AS u ON e.user_id = u.user_id
WHERE e.event_time >= '2024-06-01'
GROUP BY e.event_type, u.plan, u.country
ORDER BY events DESC
LIMIT 20;
```

## Performance Settings for Large HDFS Reads

```sql
SELECT count()
FROM hdfs_events_parquet
SETTINGS
    max_threads                 = 32,
    max_read_buffer_size        = 16777216,  -- 16 MB read buffer
    hdfs_replication            = 1;
```

## Listing HDFS Directory Contents

ClickHouse does not have a built-in HDFS `ls` command, but you can infer available files by querying a glob path that matches file metadata.

```bash
# Use hdfs dfs to list files from the shell
hdfs dfs -ls /data/events/2024/06/
```

```text
-rw-r--r--   3 hadoop hadoop  1234567890 2024-06-15 events_20240615_0.parquet
-rw-r--r--   3 hadoop hadoop  1098765432 2024-06-15 events_20240615_1.parquet
```

## Limitations

- No indexes - all HDFS reads are full file scans (Parquet column pruning applies).
- Performance depends on HDFS throughput and network bandwidth.
- No mutations or in-place updates on HDFS files.
- Kerberos authentication requires additional configuration.
- High-latency HDFS clusters will slow interactive queries; prefer pre-loading for repeated use.

## Summary

The `HDFS` engine enables ClickHouse to act as a fast query layer over Hadoop data lakes. It reads CSV, Parquet, ORC, JSON, and Avro files directly from HDFS using glob patterns and pushes column projection down for Parquet. For low-latency repeated queries, load HDFS data into a local MergeTree table. Use the `hdfs()` table function for one-off queries without creating a permanent table definition.
