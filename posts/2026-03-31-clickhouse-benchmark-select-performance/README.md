# How to Benchmark SELECT Performance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SELECT Performance, Benchmark, Query Profiling, EXPLAIN

Description: Learn how to benchmark SELECT performance in ClickHouse using EXPLAIN, query profiling, system tables, and clickhouse-benchmark for accurate measurements.

---

## Benchmarking SELECT Queries

Accurate SELECT benchmarking requires warming the page cache, running multiple iterations, and measuring p50/p95/p99 latencies rather than single runs. ClickHouse provides dedicated tools for all of this.

## Warm Up the Cache First

Always run a query once before timing it to warm OS and ClickHouse page caches:

```bash
# Warm-up run (discard result)
clickhouse-client --query "SELECT count() FROM orders" --format Null

# Timed run
time clickhouse-client --query "SELECT count() FROM orders" --format Null
```

## Using EXPLAIN to Understand the Plan

Before benchmarking, understand what ClickHouse will do:

```sql
EXPLAIN SELECT
    region,
    sum(revenue) AS total
FROM orders
WHERE order_date >= today() - 30
GROUP BY region;
```

Check index usage:

```sql
EXPLAIN indexes=1
SELECT * FROM orders WHERE order_date = today();
```

## Profiling with system.query_log

Run with query logging enabled:

```sql
SET log_queries = 1;
SELECT region, sum(revenue) FROM orders WHERE order_date >= today() - 30 GROUP BY region;
```

Then inspect:

```sql
SELECT
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%sum(revenue)%'
ORDER BY event_time DESC
LIMIT 5;
```

## Using clickhouse-benchmark for Statistical Results

```bash
clickhouse-benchmark \
    --iterations 50 \
    --concurrency 1 \
    --query "SELECT region, sum(revenue) FROM orders GROUP BY region ORDER BY sum(revenue) DESC"
```

Output includes p50/p95/p99 latencies:

```text
50.000%     12.34 ms
95.000%     18.92 ms
99.000%     24.11 ms
```

## Testing With and Without Indexes

```bash
# Without index
clickhouse-benchmark --iterations 30 \
    --query "SELECT count() FROM orders WHERE customer_id = 42345"

# Add index
clickhouse-client --query "
ALTER TABLE orders ADD INDEX idx_customer customer_id TYPE bloom_filter GRANULARITY 1;
ALTER TABLE orders MATERIALIZE INDEX idx_customer;
"

# With index
clickhouse-benchmark --iterations 30 \
    --query "SELECT count() FROM orders WHERE customer_id = 42345"
```

## Measuring the Effect of Compression Codecs

```bash
# Query the uncompressed version
clickhouse-benchmark --iterations 30 \
    --query "SELECT avg(amount) FROM orders_uncompressed"

# Query the compressed version
clickhouse-benchmark --iterations 30 \
    --query "SELECT avg(amount) FROM orders_compressed"
```

## Reading Query Metrics

```sql
SELECT
    metric,
    value
FROM system.metrics
WHERE metric IN ('Query', 'MemoryTracking', 'ReadonlyReplica')
ORDER BY metric;
```

## Summary

ClickHouse SELECT benchmarks require cache warm-up, multiple iterations via `clickhouse-benchmark`, and inspection of `system.query_log` for read_rows and memory_usage. Use `EXPLAIN indexes=1` before benchmarking to verify the query uses the expected index path.
