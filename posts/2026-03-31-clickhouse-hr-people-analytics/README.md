# How to Use ClickHouse for HR and People Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, HR Analytics, People Analytics, Workforce, Retention

Description: Use ClickHouse to analyze HR data including employee retention, hiring funnel performance, compensation equity, and workforce diversity metrics.

---

## People Analytics at Scale

Large organizations have thousands of employees and generate millions of HR system events: job applications, performance reviews, compensation changes, and engagement surveys. ClickHouse enables HR teams to run analytics across all this data and build dashboards for headcount planning, retention risk, and hiring efficiency.

## Employee Data Schema

```sql
CREATE TABLE employee_events
(
    event_date Date,
    employee_id UInt64,
    event_type LowCardinality(String),
    department LowCardinality(String),
    job_level LowCardinality(String),
    location LowCardinality(String),
    gender LowCardinality(String),
    ethnicity LowCardinality(String),
    tenure_months UInt16,
    base_salary Decimal(10, 2),
    performance_score Float32,
    manager_id UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (department, event_date, employee_id);
```

## Employee Attrition Analysis

```sql
-- Monthly attrition rate by department
SELECT
    department,
    toYYYYMM(event_date) AS month,
    countIf(event_type = 'termination') AS terminations,
    countIf(event_type = 'headcount_snapshot') AS headcount,
    round(countIf(event_type = 'termination') /
          nullIf(countIf(event_type = 'headcount_snapshot'), 0) * 100, 2) AS attrition_rate_pct
FROM employee_events
WHERE event_date >= today() - 365
GROUP BY department, month
ORDER BY department, month;
```

## Tenure Distribution at Attrition

```sql
-- When do employees typically leave?
SELECT
    department,
    multiIf(
        tenure_months < 6, '0-6 months',
        tenure_months < 12, '6-12 months',
        tenure_months < 24, '1-2 years',
        tenure_months < 48, '2-4 years',
        '4+ years'
    ) AS tenure_bucket,
    count() AS voluntary_exits
FROM employee_events
WHERE event_type = 'voluntary_termination'
  AND event_date >= today() - 730
GROUP BY department, tenure_bucket
ORDER BY department, voluntary_exits DESC;
```

## Compensation Equity Analysis

```sql
-- Pay gap analysis by gender and level
SELECT
    job_level,
    gender,
    count() AS employee_count,
    avg(base_salary) AS avg_salary,
    median(base_salary) AS median_salary,
    min(base_salary) AS min_salary,
    max(base_salary) AS max_salary
FROM employee_events
WHERE event_type = 'headcount_snapshot'
  AND event_date = (SELECT max(event_date) FROM employee_events WHERE event_type = 'headcount_snapshot')
GROUP BY job_level, gender
ORDER BY job_level, gender;
```

## Hiring Funnel Performance

```sql
CREATE TABLE recruiting_events
(
    event_date Date,
    candidate_id UInt64,
    requisition_id UInt32,
    department LowCardinality(String),
    source LowCardinality(String),
    stage LowCardinality(String),
    outcome LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_date)
ORDER BY (department, requisition_id, candidate_id, event_date);
```

```sql
-- Hiring funnel conversion rates by source
SELECT
    source,
    countIf(stage = 'applied') AS applied,
    countIf(stage = 'phone_screen') AS phone_screen,
    countIf(stage = 'onsite') AS onsite,
    countIf(stage = 'offer') AS offers,
    countIf(outcome = 'hired') AS hired,
    round(countIf(outcome = 'hired') /
          nullIf(countIf(stage = 'applied'), 0) * 100, 1) AS overall_conversion_pct
FROM recruiting_events
WHERE event_date >= today() - 180
GROUP BY source
ORDER BY hired DESC;
```

## Summary

ClickHouse enables fast, scalable HR and people analytics across employee lifecycle events, compensation data, and recruiting funnels. Build attrition analysis, tenure distribution reports, compensation equity dashboards, and hiring funnel conversion metrics using MergeTree with partitioning by month and sorting by department and date.
