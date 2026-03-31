# How to Respond to MySQL Slow Query Incidents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Slow Query, Performance, Incident Response, Index

Description: Learn how to identify, triage, and resolve MySQL slow query incidents quickly to restore database performance and application stability.

---

## Recognizing a Slow Query Incident

Slow query incidents manifest as elevated application response times, growing queue depths, or CPU spikes on the database host. MySQL defines a slow query as any query exceeding the `long_query_time` threshold (default 10 seconds, but typically set to 1-2 seconds in production).

Start by enabling the slow query log if it is not already active:

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

## Immediate Triage

Check for currently running long queries:

```sql
SELECT id, user, host, db, time, state, LEFT(info, 100) AS query_snippet
FROM information_schema.PROCESSLIST
WHERE command = 'Query'
  AND time > 5
ORDER BY time DESC;
```

If a query has been running for an unexpectedly long time and you need to stop it immediately:

```sql
KILL QUERY <process_id>;
```

Use `KILL QUERY` (not `KILL CONNECTION`) to terminate only the query, leaving the connection intact for the application.

## Analyzing the Slow Query Log

Use `mysqldumpslow` to summarize the slow query log and identify the worst offenders:

```bash
# Top 10 slowest queries by total execution time
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Top 10 queries run most frequently
mysqldumpslow -s c -t 10 /var/log/mysql/slow.log
```

For more detailed analysis, use `pt-query-digest` from the Percona Toolkit:

```bash
pt-query-digest /var/log/mysql/slow.log > /tmp/slow_query_report.txt
```

## Diagnosing Query Execution

Once you identify the slow query, run `EXPLAIN` to understand its execution plan:

```sql
EXPLAIN SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
  AND o.created_at > '2026-01-01';
```

Look for these red flags in the output:
- `type: ALL` - full table scan, no index used
- `rows` column showing millions of rows examined
- `Extra: Using filesort` or `Extra: Using temporary` - costly sort/temp operations

## Applying a Fix

For queries doing full table scans, add the appropriate index:

```sql
-- Add a composite index to cover the WHERE clause columns
ALTER TABLE orders
  ADD INDEX idx_status_created (status, created_at);

-- Verify the index is used after creation
EXPLAIN SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
  AND o.created_at > '2026-01-01';
```

For complex queries, consider rewriting with CTEs or breaking them into simpler steps.

## Performance Schema Analysis

The Performance Schema provides real-time query statistics:

```sql
SELECT digest_text, count_star, avg_timer_wait / 1e12 AS avg_latency_s,
       sum_rows_examined / count_star AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
ORDER BY avg_timer_wait DESC
LIMIT 10;
```

## Post-Incident Actions

After resolving the immediate incident, update your runbook and address the systemic issues:
- Review `long_query_time` settings and ensure slow query logging is always enabled
- Add slow query metrics to your monitoring dashboard with alerting
- Schedule regular query review sessions using `pt-query-digest` reports
- Implement query review gates in your CI/CD pipeline using tools like `sqlfluff`

## Summary

Responding to MySQL slow query incidents requires immediately identifying running slow queries via `SHOW PROCESSLIST`, analyzing the slow query log with `mysqldumpslow` or `pt-query-digest`, diagnosing execution plans with `EXPLAIN`, and resolving root causes through indexing or query rewrites. Proactive slow query logging and monitoring are essential for catching problems before they escalate.
