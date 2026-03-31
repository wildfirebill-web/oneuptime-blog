# How to Build an Audit Trail for Data Changes in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Audit Trail, Compliance, Change Tracking, MergeTree

Description: Learn how to build a comprehensive audit trail for data changes in ClickHouse to track who changed what and when for compliance and debugging.

---

## Why an Audit Trail in ClickHouse?

Audit trails record every change to sensitive data - who changed it, when, from what value, and to what value. This is critical for compliance (SOX, HIPAA, GDPR), debugging data quality issues, and understanding data lineage.

## Schema Design for an Audit Table

```sql
CREATE TABLE data_audit_log (
    event_time      DateTime,
    table_name      LowCardinality(String),
    operation       LowCardinality(String),  -- INSERT, UPDATE, DELETE
    primary_key     String,
    field_name      String,
    old_value       Nullable(String),
    new_value       Nullable(String),
    changed_by      String,
    client_ip       IPv4,
    session_id      String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, table_name, primary_key)
TTL event_time + INTERVAL 7 YEAR;
```

## Recording Changes via Application Layer

In your application, write to the audit log whenever data changes:

```python
def update_user_plan(user_id, old_plan, new_plan, changed_by):
    client.execute(
        """
        INSERT INTO data_audit_log
        (event_time, table_name, operation, primary_key,
         field_name, old_value, new_value, changed_by)
        VALUES
        """,
        [(
            datetime.utcnow(), 'users', 'UPDATE', str(user_id),
            'plan', old_plan, new_plan, changed_by
        )]
    )
```

## Tracking Mutations as Audit Events

Log ClickHouse mutations to the audit trail by capturing them from system.mutations:

```sql
SELECT
    create_time,
    table,
    command,
    parts_to_do,
    is_done
FROM system.mutations
WHERE create_time >= now() - INTERVAL 7 DAY
ORDER BY create_time DESC;
```

## Query: Full Change History for a Record

```sql
SELECT
    event_time,
    operation,
    field_name,
    old_value,
    new_value,
    changed_by
FROM data_audit_log
WHERE table_name = 'users'
  AND primary_key = '42'
ORDER BY event_time;
```

## Query: Recent Changes by User

```sql
SELECT
    changed_by,
    table_name,
    operation,
    count() AS change_count,
    max(event_time) AS last_change
FROM data_audit_log
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY changed_by, table_name, operation
ORDER BY change_count DESC;
```

## Query: Detecting Unusual Deletion Patterns

```sql
SELECT
    changed_by,
    count() AS deletions,
    uniq(primary_key) AS distinct_records
FROM data_audit_log
WHERE operation = 'DELETE'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY changed_by
HAVING deletions > 1000
ORDER BY deletions DESC;
```

## Using ClickHouse Access Log for System-Level Audit

Enable query logging for system-level audit:

```xml
<query_log>
    <database>system</database>
    <table>query_log</table>
    <flush_interval_milliseconds>7500</flush_interval_milliseconds>
</query_log>
```

```sql
SELECT user, query_kind, query, event_time, client_hostname
FROM system.query_log
WHERE query_kind IN ('Insert', 'Delete')
  AND event_date = today()
ORDER BY event_time DESC;
```

## Summary

A robust ClickHouse audit trail combines application-layer change logging into a dedicated audit table, system.query_log monitoring for DDL and mutation tracking, and long-retention TTL policies to satisfy compliance requirements. Design the audit table with partitioning for efficient time-range queries and set TTL to match your regulatory retention obligations.
