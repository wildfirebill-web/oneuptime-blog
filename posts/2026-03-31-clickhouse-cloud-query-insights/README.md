# How to Use ClickHouse Cloud Query Insights

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Cloud, Query Insights, Performance, Monitoring, Observability

Description: Learn how to use ClickHouse Cloud Query Insights to identify slow queries, analyze resource usage, and optimize query performance in production.

---

ClickHouse Cloud Query Insights is a built-in observability feature that provides a visual interface for analyzing query performance, resource consumption, and error patterns - without needing to query `system.query_log` directly.

## Accessing Query Insights

In the ClickHouse Cloud console:
1. Open your service
2. Click "Monitoring" - "Query Insights"
3. Set your time range and explore the dashboard

## Key Metrics Displayed

- **Query duration distribution**: Histogram of query execution times
- **Top slow queries**: Ranked by average or max duration
- **Resource consumers**: Queries by memory usage, rows read, and bytes read
- **Error summary**: Failed queries grouped by error type
- **User breakdown**: Activity per database user

## Identifying Slow Queries

The "Slowest Queries" view shows queries sorted by P99 latency. Click any query to see:
- Full query text
- Execution plan
- Resource usage breakdown
- Call frequency

## Analyzing via SQL Directly

For custom analysis, query the underlying data:

```sql
SELECT
    normalizeQuery(query) AS query_pattern,
    count() AS executions,
    avg(query_duration_ms) AS avg_ms,
    quantile(0.99)(query_duration_ms) AS p99_ms,
    sum(read_bytes) AS total_bytes_read
FROM system.query_log
WHERE event_time > now() - INTERVAL 24 HOUR
  AND type = 'QueryFinish'
  AND user != 'system'
GROUP BY query_pattern
ORDER BY p99_ms DESC
LIMIT 20;
```

## Finding Memory-Heavy Queries

```sql
SELECT
    query_id,
    user,
    formatReadableSize(memory_usage) AS memory,
    query_duration_ms,
    substr(query, 1, 100) AS query_preview
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR
  AND type = 'QueryFinish'
ORDER BY memory_usage DESC
LIMIT 10;
```

## Tracking Error Rates

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    countIf(type = 'ExceptionBeforeStart') AS errors_before,
    countIf(type = 'ExceptionWhileProcessing') AS errors_during,
    count() AS total
FROM system.query_log
WHERE event_time > now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour;
```

## Setting Up Alerts

In ClickHouse Cloud console, configure alerts based on Query Insights metrics:
- P99 latency above threshold
- Error rate exceeding X%
- Memory usage spikes

## Summary

ClickHouse Cloud Query Insights provides a visual, no-SQL-required interface for query performance analysis. Use it to identify slow queries, memory consumers, and error patterns. Combine it with direct `system.query_log` queries for deeper custom analysis and alerting.
