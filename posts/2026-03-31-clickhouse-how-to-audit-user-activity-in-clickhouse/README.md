# How to Audit User Activity in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Auditing, Security, Compliance, Query Log

Description: Set up comprehensive user activity auditing in ClickHouse using query logs, session logs, and access log tables for security and compliance.

---

## Why Audit User Activity?

For compliance (GDPR, SOC2, HIPAA) and security operations, you need to know:
- Who ran which queries and when
- Which data was accessed
- Failed login attempts
- Schema changes (DDL operations)
- Users with elevated privileges

ClickHouse provides built-in logging tables in the `system` database that record all this information.

## Enabling Query Logging

By default, ClickHouse logs queries to `system.query_log`. Ensure it is enabled:

```xml
<!-- config.xml -->
<query_log>
    <database>system</database>
    <table>query_log</table>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    <ttl>event_date + INTERVAL 90 DAY</ttl>
</query_log>
```

Verify it is active:

```sql
SELECT count() FROM system.query_log WHERE event_time > now() - INTERVAL 1 HOUR;
```

## Auditing Query Activity

### Find All Queries by a Specific User

```sql
SELECT
    event_time,
    query_id,
    type,
    query_duration_ms,
    read_rows,
    left(query, 300) AS query_preview
FROM system.query_log
WHERE user = 'analyst_alice'
  AND event_time > now() - INTERVAL 7 DAY
  AND type IN ('QueryFinish', 'ExceptionWhileProcessing')
ORDER BY event_time DESC;
```

### Audit DDL Operations

```sql
-- Find all CREATE, DROP, ALTER, RENAME operations
SELECT
    event_time,
    user,
    query_kind,
    query
FROM system.query_log
WHERE query_kind IN ('Create', 'Drop', 'Alter', 'Rename', 'Truncate')
  AND event_time > now() - INTERVAL 30 DAY
ORDER BY event_time DESC;
```

### Find Failed Authentication Attempts

```sql
-- Query failures that look like auth issues
SELECT
    event_time,
    user,
    exception_code,
    exception,
    client_hostname,
    client_name
FROM system.query_log
WHERE type = 'ExceptionBeforeStart'
  AND exception LIKE '%Authentication%'
  AND event_time > now() - INTERVAL 7 DAY
ORDER BY event_time DESC;
```

### Track Data Exports

```sql
-- Identify queries that may have exported large datasets
SELECT
    event_time,
    user,
    result_rows,
    result_bytes,
    client_hostname,
    left(query, 200) AS q
FROM system.query_log
WHERE type = 'QueryFinish'
  AND result_rows > 1000000
  AND event_time > now() - INTERVAL 7 DAY
ORDER BY result_rows DESC;
```

## Session Log

ClickHouse also logs session-level events (login/logout):

```sql
-- See user login activity
SELECT
    event_time,
    user,
    client_hostname,
    client_port,
    interface,
    type
FROM system.session_log
WHERE event_time > now() - INTERVAL 7 DAY
ORDER BY event_time DESC
LIMIT 100;
```

Enable session log in config:

```xml
<session_log>
    <database>system</database>
    <table>session_log</table>
    <ttl>event_date + INTERVAL 90 DAY</ttl>
</session_log>
```

## Building an Audit Dashboard Query

```sql
-- Daily activity summary per user
SELECT
    toDate(event_time) AS day,
    user,
    countIf(type = 'QueryFinish') AS successful_queries,
    countIf(type LIKE 'Exception%') AS failed_queries,
    sum(read_rows) AS total_rows_read,
    formatReadableSize(sum(read_bytes)) AS total_data_read,
    count(DISTINCT client_hostname) AS distinct_hosts
FROM system.query_log
WHERE event_time > now() - INTERVAL 30 DAY
GROUP BY day, user
ORDER BY day DESC, total_rows_read DESC;
```

## Exporting Audit Logs to External Systems

```bash
#!/bin/bash
# Export daily audit log to S3 for long-term retention
DATE=$(date -d "yesterday" +%Y-%m-%d)
clickhouse-client --query "
SELECT
    event_time,
    user,
    query_kind,
    exception_code,
    client_hostname,
    query
FROM system.query_log
WHERE toDate(event_time) = '$DATE'
FORMAT CSVWithNames
" | aws s3 cp - "s3://audit-bucket/clickhouse-logs/$DATE/query_log.csv"
echo "Exported audit log for $DATE"
```

## Setting Up Alerts for Suspicious Activity

```sql
-- Find users querying sensitive tables
SELECT
    event_time,
    user,
    client_hostname,
    left(query, 200) AS q
FROM system.query_log
WHERE type = 'QueryFinish'
  AND (query LIKE '%user_pii%' OR query LIKE '%payments%' OR query LIKE '%credentials%')
  AND user NOT IN ('etl_service', 'analytics_trusted')
  AND event_time > now() - INTERVAL 24 HOUR
ORDER BY event_time DESC;
```

## Summary

ClickHouse provides comprehensive user activity auditing through `system.query_log` and `system.session_log`. Enable both with appropriate TTL settings in `config.xml`, then use SQL queries to audit who ran what, when, and from where. Build daily aggregated audit reports for compliance dashboards, export to S3 for long-term retention, and set up alert queries for suspicious access patterns like large data exports or access to sensitive tables by non-privileged users.
