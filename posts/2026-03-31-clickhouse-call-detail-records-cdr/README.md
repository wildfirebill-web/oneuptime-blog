# How to Store and Analyze CDR (Call Detail Records) in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CDR, Telecom, Call Detail Record, Network Analytics

Description: Store and analyze call detail records in ClickHouse for telecom analytics including call duration, routing, and quality metrics.

---

Call Detail Records (CDRs) are the lifeblood of telecom analytics. Every call, SMS, and data session generates a record that must be stored for billing, quality assurance, and regulatory compliance. ClickHouse handles the extreme volume of CDR data with efficient compression and fast query performance.

## CDR Table Schema

```sql
CREATE TABLE cdr (
    call_start      DateTime,
    call_end        DateTime,
    duration_s      UInt32,
    caller_number   String,
    callee_number   String,
    caller_imsi     String,
    callee_imsi     String,
    call_type       LowCardinality(String),
    network_type    LowCardinality(String),
    origin_cell     String,
    terminating_cell String,
    disconnect_cause UInt16,
    mo_country      LowCardinality(String),
    mt_country      LowCardinality(String),
    charge_amount   Float32,
    date            Date DEFAULT toDate(call_start)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (caller_imsi, call_start);
```

## Call Volume by Hour

```sql
SELECT
    toStartOfHour(call_start) AS hour,
    network_type,
    count() AS total_calls,
    sum(duration_s) / 60 AS total_minutes
FROM cdr
WHERE date = today()
GROUP BY hour, network_type
ORDER BY hour;
```

## Average Call Duration by Call Type

```sql
SELECT
    call_type,
    count() AS calls,
    round(avg(duration_s) / 60, 2) AS avg_duration_min,
    quantile(0.95)(duration_s) / 60 AS p95_duration_min
FROM cdr
WHERE date >= today() - 7
GROUP BY call_type
ORDER BY calls DESC;
```

## Top Destinations by Volume

```sql
SELECT
    mt_country,
    count() AS call_count,
    sum(duration_s) AS total_seconds,
    round(sum(charge_amount), 2) AS total_revenue
FROM cdr
WHERE call_type = 'international'
  AND date >= today() - 30
GROUP BY mt_country
ORDER BY call_count DESC
LIMIT 20;
```

## Disconnect Cause Analysis

Identify the most common call failure reasons:

```sql
SELECT
    disconnect_cause,
    count() AS occurrences,
    round(count() / sum(count()) OVER () * 100, 2) AS pct
FROM cdr
WHERE duration_s < 5
  AND date = today()
GROUP BY disconnect_cause
ORDER BY occurrences DESC
LIMIT 20;
```

## Revenue by Subscriber

```sql
SELECT
    caller_imsi,
    count() AS calls,
    round(sum(charge_amount), 2) AS total_spend,
    round(sum(duration_s) / 3600, 2) AS total_hours
FROM cdr
WHERE date >= today() - 30
GROUP BY caller_imsi
ORDER BY total_spend DESC
LIMIT 50;
```

## Summary

ClickHouse stores CDR data with exceptional compression ratios - telecom records are highly repetitive and compress well. By partitioning by date and ordering by IMSI, individual subscriber queries remain fast even with years of history, while aggregate queries over the full dataset complete in seconds.
