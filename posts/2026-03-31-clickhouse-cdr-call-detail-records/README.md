# How to Store and Analyze CDR (Call Detail Records) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Telecom, CDR, Call Detail Record, Analytics

Description: Learn how to ingest and query telecom Call Detail Records in ClickHouse for billing validation, usage analysis, and regulatory compliance.

---

Call Detail Records are the raw transaction logs of a telecom network. Each voice call, SMS, or data session generates a CDR. With millions of records per hour across a carrier's network, an analytical database like ClickHouse is essential for fast reporting and compliance.

## CDR Table Schema

```sql
CREATE TABLE cdr (
    cdr_id          UUID,
    switch_id       UInt32,
    a_number        String,           -- calling party
    b_number        String,           -- called party
    call_start      DateTime,
    call_end        DateTime,
    duration_secs   UInt32,
    call_type       LowCardinality(String),  -- 'voice', 'sms', 'data'
    direction       LowCardinality(String),  -- 'inbound', 'outbound', 'roaming'
    network_type    LowCardinality(String),  -- '4G', '5G', 'VoLTE'
    termination     LowCardinality(String),  -- 'normal', 'busy', 'no_answer', 'failed'
    bytes_up        UInt64,
    bytes_down      UInt64,
    cell_id         UInt32,
    cost_cents      UInt32
) ENGINE = MergeTree()
ORDER BY (switch_id, call_start)
PARTITION BY toYYYYMM(call_start);
```

## Total Usage by Call Type per Day

```sql
SELECT
    toDate(call_start)     AS day,
    call_type,
    count()                AS total_calls,
    sum(duration_secs)     AS total_secs,
    sum(cost_cents) / 100  AS total_revenue
FROM cdr
WHERE call_start >= today() - 30
GROUP BY day, call_type
ORDER BY day, call_type;
```

## Top Originating Numbers by Volume

Useful for detecting bulk dialers or fraudulent sources.

```sql
SELECT
    a_number,
    count()            AS call_count,
    sum(duration_secs) AS total_secs
FROM cdr
WHERE call_type = 'voice'
  AND call_start >= today() - 1
GROUP BY a_number
ORDER BY call_count DESC
LIMIT 20;
```

## Termination Code Distribution

```sql
SELECT
    termination,
    count()                                                AS calls,
    round(100.0 * count() / sum(count()) OVER (), 2)       AS pct
FROM cdr
WHERE call_start >= today() - 7
GROUP BY termination
ORDER BY calls DESC;
```

## Data Usage by Network Type

```sql
SELECT
    network_type,
    count()                             AS sessions,
    sum(bytes_up + bytes_down) / 1e9    AS total_gb,
    avg(bytes_up + bytes_down) / 1e6    AS avg_mb_per_session
FROM cdr
WHERE call_type = 'data'
  AND call_start >= today() - 7
GROUP BY network_type
ORDER BY total_gb DESC;
```

## Hourly Call Volume Heatmap

```sql
SELECT
    toDayOfWeek(call_start)    AS weekday,
    toHour(call_start)         AS hour,
    count()                    AS calls
FROM cdr
WHERE call_start >= today() - 28
GROUP BY weekday, hour
ORDER BY weekday, hour;
```

## Long-Duration Anomaly Detection

Calls lasting many hours can indicate toll fraud or misconfigured equipment.

```sql
SELECT
    a_number,
    b_number,
    call_start,
    duration_secs / 3600 AS duration_hours
FROM cdr
WHERE duration_secs > 7200   -- longer than 2 hours
  AND call_type = 'voice'
  AND call_start >= today() - 1
ORDER BY duration_secs DESC
LIMIT 50;
```

## Summary

ClickHouse excels as a CDR analytics store because its columnar compression dramatically reduces storage for repetitive telecom data, and its vectorized aggregation handles billions of records in seconds. Partition by month for efficient retention management, use `LowCardinality` for categorical fields, and layer materialized views on top for instant billing dashboards.
