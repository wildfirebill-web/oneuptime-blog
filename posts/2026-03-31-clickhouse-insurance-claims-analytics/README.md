# How to Use ClickHouse for Insurance Claims Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Insurance, Claims Analytics, Fraud Detection, Actuarial

Description: Use ClickHouse to analyze insurance claims at scale, detect fraud patterns, calculate loss ratios, and build actuarial reporting pipelines.

---

## Why ClickHouse for Insurance Claims

Insurance companies process millions of claims per year. Actuarial analysis, fraud detection, and regulatory reporting require scanning large historical datasets with complex aggregations - exactly what ClickHouse excels at. Queries that take hours in traditional databases complete in seconds with ClickHouse.

## Claims Data Schema

```sql
CREATE TABLE insurance_claims
(
    claim_id UInt64,
    policy_id UInt64,
    claim_date Date,
    claim_type LowCardinality(String),
    line_of_business LowCardinality(String),
    state LowCardinality(String),
    reported_amount Decimal(12, 2),
    paid_amount Decimal(12, 2),
    reserved_amount Decimal(12, 2),
    status LowCardinality(String),
    claimant_age UInt8,
    provider_id UInt64,
    adjuster_id UInt32,
    days_to_close UInt16,
    fraud_score Float32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(claim_date)
ORDER BY (line_of_business, state, claim_date);
```

## Loss Ratio Analysis

Loss ratio is the key profitability metric: claims paid divided by premiums earned.

```sql
SELECT
    line_of_business,
    toYear(claim_date) AS year,
    count() AS claim_count,
    sum(paid_amount) AS total_paid,
    sum(reported_amount) AS total_incurred,
    -- Loss ratio would need premium data joined here
    avg(paid_amount) AS avg_claim_size,
    quantile(0.95)(paid_amount) AS p95_claim_size,
    countIf(paid_amount > 100000) AS large_loss_count
FROM insurance_claims
WHERE status = 'Closed'
GROUP BY line_of_business, year
ORDER BY line_of_business, year;
```

## Fraud Detection Signals

```sql
-- Providers with abnormally high claim frequency
SELECT
    provider_id,
    count() AS claim_count,
    avg(paid_amount) AS avg_paid,
    avg(fraud_score) AS avg_fraud_score,
    countIf(fraud_score > 0.8) AS high_risk_count
FROM insurance_claims
WHERE claim_date >= today() - 365
GROUP BY provider_id
HAVING claim_count > 50
   AND avg_fraud_score > 0.5
ORDER BY avg_fraud_score DESC
LIMIT 20;
```

```sql
-- Claimants with multiple claims in short windows
SELECT
    policy_id,
    count() AS claims_in_90_days,
    sum(reported_amount) AS total_reported
FROM insurance_claims
WHERE claim_date >= today() - 90
GROUP BY policy_id
HAVING claims_in_90_days >= 3
ORDER BY total_reported DESC;
```

## Claims Aging Report

```sql
SELECT
    status,
    multiIf(
        days_to_close <= 30, '0-30 days',
        days_to_close <= 60, '31-60 days',
        days_to_close <= 90, '61-90 days',
        '90+ days'
    ) AS aging_bucket,
    count() AS claim_count,
    avg(paid_amount) AS avg_paid,
    sum(reserved_amount) AS total_reserves
FROM insurance_claims
WHERE claim_date >= today() - 365
GROUP BY status, aging_bucket
ORDER BY status, aging_bucket;
```

## Reserve Adequacy Tracking

```sql
-- Track how reserves compare to actual payments
SELECT
    line_of_business,
    toYYYYMM(claim_date) AS month,
    sum(reserved_amount) AS total_reserved,
    sum(paid_amount) AS total_paid,
    round(sum(paid_amount) / nullIf(sum(reserved_amount), 0) * 100, 1) AS reserve_utilization_pct
FROM insurance_claims
WHERE status = 'Closed'
GROUP BY line_of_business, month
ORDER BY line_of_business, month;
```

## Summary

ClickHouse handles insurance claims analytics with ease - loss ratio calculations, fraud signal detection, aging analysis, and reserve adequacy reports all run in seconds on millions of claims. Model your claims data partitioned by month and sorted by line of business and date for optimal query performance across actuarial and operational reports.
