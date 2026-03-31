# How to Track Data Access Patterns for Compliance in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Compliance, Audit, Data Access, GDPR

Description: Track data access patterns in ClickHouse for compliance with GDPR, HIPAA, and SOC 2 by logging and analyzing who accesses what data.

---

Compliance requirements like GDPR, HIPAA, and SOC 2 demand that you know who accessed sensitive data and when. ClickHouse can serve as a centralized audit log store, providing the query capabilities needed for compliance reporting and anomaly detection.

## Data Access Audit Log Table

```sql
CREATE TABLE data_access_logs
(
    log_id UUID DEFAULT generateUUIDv4(),
    user_id UInt64,
    user_name String,
    user_role LowCardinality(String),
    data_category LowCardinality(String),
    resource_type LowCardinality(String),
    resource_id String,
    action LowCardinality(String),
    outcome LowCardinality(String),
    source_ip IPv4,
    application LowCardinality(String),
    record_count UInt32 DEFAULT 0,
    accessed_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(accessed_at)
ORDER BY (data_category, user_id, accessed_at)
TTL toDate(accessed_at) + INTERVAL 2555 DAY;
```

## Access Summary by Data Category

```sql
SELECT
    data_category,
    action,
    count() AS access_count,
    countDistinct(user_id) AS unique_users,
    sum(record_count) AS total_records_accessed
FROM data_access_logs
WHERE accessed_at >= now() - INTERVAL 30 DAY
GROUP BY data_category, action
ORDER BY total_records_accessed DESC;
```

## User Access Report for a Specific Period

Useful for compliance audits and individual data subject requests.

```sql
SELECT
    user_name,
    user_role,
    data_category,
    resource_type,
    action,
    count() AS access_count,
    sum(record_count) AS records_touched,
    min(accessed_at) AS first_access,
    max(accessed_at) AS last_access
FROM data_access_logs
WHERE user_id = 42
  AND accessed_at BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY user_name, user_role, data_category, resource_type, action
ORDER BY last_access DESC;
```

## Bulk Data Exports - Potential Data Exfiltration

```sql
SELECT
    user_id,
    user_name,
    data_category,
    sum(record_count) AS records_exported,
    count() AS export_operations,
    min(accessed_at) AS start_time,
    max(accessed_at) AS end_time
FROM data_access_logs
WHERE action = 'export'
  AND accessed_at >= now() - INTERVAL 24 HOUR
GROUP BY user_id, user_name, data_category
HAVING records_exported > 10000
ORDER BY records_exported DESC;
```

## Access Outside Business Hours

```sql
SELECT
    user_name,
    user_role,
    data_category,
    action,
    source_ip,
    accessed_at,
    toHour(toTimeZone(accessed_at, 'UTC')) AS utc_hour
FROM data_access_logs
WHERE accessed_at >= now() - INTERVAL 7 DAY
  AND (toHour(toTimeZone(accessed_at, 'UTC')) < 8
       OR toHour(toTimeZone(accessed_at, 'UTC')) > 20)
  AND data_category IN ('PII', 'PHI', 'financial')
ORDER BY accessed_at DESC;
```

## Access to Specific Subject's Data (GDPR Subject Request)

```sql
SELECT
    user_name,
    user_role,
    action,
    application,
    record_count,
    accessed_at
FROM data_access_logs
WHERE resource_id = 'user_id:98765'
  AND data_category = 'PII'
ORDER BY accessed_at DESC;
```

## Monthly Compliance Summary

```sql
SELECT
    toStartOfMonth(accessed_at) AS month,
    data_category,
    count() AS access_events,
    countDistinct(user_id) AS unique_users,
    sum(record_count) AS total_records,
    countIf(outcome = 'denied') AS denied_attempts
FROM data_access_logs
WHERE accessed_at >= now() - INTERVAL 6 MONTH
GROUP BY month, data_category
ORDER BY month DESC, data_category;
```

## Summary

ClickHouse provides long-term audit log storage with a 7-year TTL (2555 days) suitable for regulatory retention requirements. Fast queries support GDPR subject access requests, bulk exfiltration detection, and scheduled compliance reports. Partitioning by month keeps historical queries efficient, and the columnar format compresses audit logs significantly compared to row-oriented databases.
