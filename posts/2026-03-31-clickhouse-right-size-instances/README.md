# How to Right-Size ClickHouse Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Performance, Cost Optimization, Infrastructure, Resource Management

Description: Learn how to right-size ClickHouse instances by analyzing query patterns, memory usage, and CPU utilization to reduce costs without sacrificing performance.

---

## Why Right-Sizing Matters

Over-provisioned ClickHouse instances waste money. Under-provisioned ones cause slow queries and failed inserts. Right-sizing means matching hardware to actual workload characteristics - query volume, data size, concurrency, and insert rates.

## Analyze Current Resource Usage

Start by examining actual consumption through system tables:

```sql
-- CPU and memory usage over the last hour
SELECT
    toStartOfMinute(event_time) AS minute,
    avg(ProfileEvents['UserTimeMicroseconds']) / 1e6 AS cpu_user_sec,
    avg(memory_usage) AS avg_memory_bytes,
    max(memory_usage) AS peak_memory_bytes
FROM system.query_log
WHERE event_time >= now() - INTERVAL 1 HOUR
  AND type = 'QueryFinish'
GROUP BY minute
ORDER BY minute;
```

```sql
-- Identify memory-intensive queries
SELECT
    query_id,
    query,
    memory_usage,
    read_rows,
    read_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 24 HOUR
ORDER BY memory_usage DESC
LIMIT 20;
```

## Memory Sizing Guidelines

ClickHouse uses memory for:
- Mark caches (accelerate reads)
- Uncompressed caches
- Query execution (aggregations, joins, sorts)
- Background merges

A practical formula for memory sizing:

```text
Required RAM = (hot dataset size * 0.1) + (concurrent queries * avg query memory) + 4GB OS overhead
```

For a server with 500GB of hot data and 10 concurrent queries averaging 2GB each:

```text
RAM = (500GB * 0.1) + (10 * 2GB) + 4GB = 50 + 20 + 4 = 74GB -> use 96GB server
```

## CPU Sizing

Check actual CPU utilization:

```bash
# On the ClickHouse host
top -b -n 5 -d 2 | grep clickhouse
```

```sql
-- CPU time by query type
SELECT
    substring(query, 1, 80) AS query_prefix,
    count() AS query_count,
    sum(ProfileEvents['UserTimeMicroseconds']) / 1e6 AS total_cpu_sec,
    avg(ProfileEvents['UserTimeMicroseconds']) / 1e6 AS avg_cpu_sec
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY query_prefix
ORDER BY total_cpu_sec DESC
LIMIT 10;
```

ClickHouse is highly parallelized. For OLAP queries, more cores directly improve performance. A typical guideline: 1 CPU core per 1-2 GB/s of sustained read throughput from disk.

## Disk Sizing

```sql
-- Current compressed and uncompressed size per table
SELECT
    table,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS ratio
FROM system.parts
WHERE active
GROUP BY table
ORDER BY sum(data_compressed_bytes) DESC;
```

Plan for 3x current data size to accommodate growth, plus temp space for large queries.

## Downsizing Safely

Before downsizing, run a load test:

```bash
clickhouse-benchmark --host=localhost --port=9000 \
  --concurrency=20 --iterations=1000 \
  --query="SELECT count() FROM hits WHERE date = today()"
```

Check that p99 latency stays within acceptable bounds and memory usage stays below 80% of the target instance's RAM.

## Summary

Right-sizing ClickHouse requires measuring actual CPU, memory, and disk usage through system tables and OS metrics. Use the memory formula to estimate RAM needs, target 60-70% average CPU utilization, and size disk at 3x current data. Validate sizing decisions with load tests before committing to smaller hardware.
