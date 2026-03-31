# How to Generate Data Quality Reports from ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Quality, Null Check, Completeness, Validation

Description: Learn how to generate data quality reports from ClickHouse to detect nulls, duplicates, out-of-range values, and schema inconsistencies at scale.

---

## Why Data Quality Reporting Matters

Poor data quality causes incorrect analytics, missed alerts, and failed ML models. ClickHouse can scan billions of rows quickly to produce data quality scorecards covering completeness, uniqueness, validity, and consistency.

## Sample Table

```sql
CREATE TABLE customer_events
(
    event_id UInt64,
    user_id UInt64,
    email String,
    event_type String,
    amount Float64,
    event_time DateTime
)
ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

## Completeness Check - Null and Empty Values

```sql
SELECT
    count() AS total_rows,
    countIf(user_id = 0) AS missing_user_id,
    countIf(email = '' OR email IS NULL) AS missing_email,
    countIf(event_type = '') AS missing_event_type,
    countIf(isNaN(amount)) AS nan_amount,
    countIf(event_time = toDateTime(0)) AS missing_event_time,
    round(countIf(user_id = 0) * 100.0 / count(), 2) AS user_id_null_pct
FROM customer_events;
```

## Uniqueness Check - Duplicate Detection

```sql
SELECT
    count() AS total_rows,
    uniq(event_id) AS unique_event_ids,
    count() - uniq(event_id) AS duplicate_count,
    round((count() - uniq(event_id)) * 100.0 / count(), 4) AS duplicate_pct
FROM customer_events;
```

Find specific duplicates:

```sql
SELECT event_id, count() AS occurrences
FROM customer_events
GROUP BY event_id
HAVING occurrences > 1
ORDER BY occurrences DESC
LIMIT 20;
```

## Validity Check - Range and Format Validation

```sql
SELECT
    countIf(amount < 0) AS negative_amounts,
    countIf(amount > 100000) AS suspiciously_large,
    countIf(event_time < '2000-01-01') AS historical_events,
    countIf(event_time > now() + INTERVAL 1 DAY) AS future_events,
    countIf(NOT match(email, '^[^@]+@[^@]+\.[^@]+$')) AS invalid_emails
FROM customer_events;
```

## Allowed Values Check

```sql
SELECT
    event_type,
    count() AS occurrences
FROM customer_events
WHERE event_type NOT IN ('page_view', 'click', 'purchase', 'signup')
GROUP BY event_type
ORDER BY occurrences DESC;
```

## Freshness Check

```sql
SELECT
    max(event_time) AS latest_event,
    now() - max(event_time) AS lag_seconds,
    if(now() - max(event_time) > 3600, 'STALE', 'FRESH') AS freshness_status
FROM customer_events;
```

## Combined Data Quality Scorecard

```sql
SELECT
    'completeness' AS dimension,
    round((1 - countIf(user_id = 0) / count()) * 100, 2) AS score
FROM customer_events
UNION ALL
SELECT
    'uniqueness',
    round(uniq(event_id) * 100.0 / count(), 2)
FROM customer_events
UNION ALL
SELECT
    'validity',
    round(countIf(amount >= 0 AND event_time > '2020-01-01') * 100.0 / count(), 2)
FROM customer_events;
```

## Summary

ClickHouse data quality reports use `countIf`, `uniq`, `isNaN`, `match`, and window scans to measure completeness, uniqueness, and validity. Combine checks into a scorecard query with UNION ALL to produce a single quality score per dimension.
