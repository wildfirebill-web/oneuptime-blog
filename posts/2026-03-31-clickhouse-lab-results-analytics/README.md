# How to Build Lab Results Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Healthcare, Laboratory, Lab Result, Diagnostic Analytics

Description: Analyze clinical lab result volumes, turnaround times, critical value rates, and test utilization trends in ClickHouse to improve laboratory operations.

---

Clinical laboratories process millions of test orders each year. Operational analytics over lab results data helps laboratory directors track turnaround times, identify critical value rates, optimize test utilization, and meet regulatory reporting requirements. ClickHouse handles this aggregated analytical workload efficiently.

## Lab Results Table

```sql
CREATE TABLE lab_results (
    result_id       UUID,
    order_id        UInt64,
    facility_id     UInt32,
    lab_id          UInt32,
    ordered_at      DateTime,
    collected_at    DateTime,
    resulted_at     DateTime,
    loinc_code      LowCardinality(String),
    test_name       LowCardinality(String),
    specimen_type   LowCardinality(String),
    result_value    Nullable(Float64),
    result_text     String,
    unit            LowCardinality(String),
    normal_low      Nullable(Float64),
    normal_high     Nullable(Float64),
    abnormal_flag   LowCardinality(String),   -- 'N', 'H', 'L', 'HH', 'LL', 'A'
    is_critical     UInt8,
    is_cancelled    UInt8
) ENGINE = MergeTree()
ORDER BY (lab_id, loinc_code, resulted_at)
PARTITION BY toYYYYMM(resulted_at);
```

## Average Turnaround Time by Test

```sql
SELECT
    test_name,
    count()                                                                     AS results,
    round(avg(dateDiff('minute', ordered_at, resulted_at)), 1)                  AS avg_tat_minutes,
    round(quantile(0.95)(dateDiff('minute', ordered_at, resulted_at)), 1)       AS p95_tat_minutes
FROM lab_results
WHERE resulted_at >= today() - 30
  AND is_cancelled = 0
GROUP BY test_name
ORDER BY avg_tat_minutes DESC
LIMIT 20;
```

## Critical Value Rate by Test

```sql
SELECT
    test_name,
    count()                                          AS results,
    countIf(is_critical = 1)                         AS critical_count,
    round(100.0 * countIf(is_critical = 1) / count(), 3) AS critical_rate_pct
FROM lab_results
WHERE resulted_at >= today() - 30
  AND is_cancelled = 0
GROUP BY test_name
HAVING results > 100
ORDER BY critical_rate_pct DESC
LIMIT 20;
```

## Daily Test Volume Trend

```sql
SELECT
    toDate(resulted_at)     AS day,
    test_name,
    count()                 AS results
FROM lab_results
WHERE resulted_at >= today() - 30
  AND is_cancelled = 0
GROUP BY day, test_name
ORDER BY day, results DESC;
```

## Abnormal Result Rate by Facility

```sql
SELECT
    facility_id,
    count()                                               AS results,
    countIf(abnormal_flag NOT IN ('', 'N'))               AS abnormal,
    round(100.0 * countIf(abnormal_flag NOT IN ('', 'N')) / count(), 2) AS abnormal_pct
FROM lab_results
WHERE resulted_at >= today() - 30
  AND is_cancelled = 0
GROUP BY facility_id
ORDER BY abnormal_pct DESC;
```

## Specimen Collection to Result Time

```sql
SELECT
    specimen_type,
    round(avg(dateDiff('minute', collected_at, resulted_at)), 1) AS avg_processing_mins,
    count() AS results
FROM lab_results
WHERE resulted_at >= today() - 30
  AND is_cancelled = 0
GROUP BY specimen_type
ORDER BY avg_processing_mins DESC;
```

## Reference Range Outlier Distribution

```sql
SELECT
    test_name,
    countIf(result_value > normal_high * 2)   AS extreme_high,
    countIf(result_value < normal_low / 2)    AS extreme_low,
    count()                                   AS total
FROM lab_results
WHERE resulted_at >= today() - 30
  AND normal_high IS NOT NULL
  AND normal_low IS NOT NULL
  AND result_value IS NOT NULL
GROUP BY test_name
ORDER BY (extreme_high + extreme_low) DESC
LIMIT 20;
```

## Summary

ClickHouse brings analytical speed to laboratory information system data. Turnaround time percentiles, critical value rates, and test utilization trends all run in seconds over millions of result records. Using `dateDiff` for time calculations and `quantile` for p95 TAT analysis gives laboratory directors the operational insight needed to meet accreditation standards and improve patient care.
