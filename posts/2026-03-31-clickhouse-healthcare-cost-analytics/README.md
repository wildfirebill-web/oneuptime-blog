# How to Build Healthcare Cost Analytics with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Healthcare, Cost Analytics, Revenue Cycle, Claims

Description: Analyze healthcare costs and claims data in ClickHouse to track cost-per-case, payer mix, denial rates, and revenue cycle performance at scale.

---

Healthcare finance teams need fast access to cost and claims data to manage margins, negotiate payer contracts, and reduce revenue leakage. ClickHouse can process millions of claim lines quickly, enabling finance and operations teams to run complex cost analyses on demand.

## Claims Table

```sql
CREATE TABLE claims (
    claim_id            UUID,
    encounter_id        UInt64,
    facility_id         UInt32,
    service_date        Date,
    billed_date         Date,
    payer_id            UInt32,
    payer_type          LowCardinality(String),  -- 'medicare', 'medicaid', 'commercial', 'self_pay'
    drg_code            LowCardinality(String),
    primary_icd10       LowCardinality(String),
    billed_amount       UInt64,   -- cents
    allowed_amount      UInt64,
    paid_amount         UInt64,
    patient_liability   UInt64,
    denial_reason       LowCardinality(String),
    claim_status        LowCardinality(String),  -- 'paid', 'denied', 'pending', 'appealed'
    los_days            UInt16
) ENGINE = MergeTree()
ORDER BY (facility_id, service_date)
PARTITION BY toYYYYMM(service_date);
```

## Net Revenue by Payer Type

```sql
SELECT
    payer_type,
    count()                                AS claims,
    round(sum(billed_amount) / 1e6, 2)    AS billed_m,
    round(sum(paid_amount) / 1e6, 2)      AS paid_m,
    round(100.0 * sum(paid_amount) / sum(billed_amount), 2) AS collection_rate_pct
FROM claims
WHERE service_date >= today() - 365
  AND claim_status = 'paid'
GROUP BY payer_type
ORDER BY paid_m DESC;
```

## Denial Rate by Payer

```sql
SELECT
    payer_id,
    payer_type,
    count()                                              AS claims_submitted,
    countIf(claim_status = 'denied')                     AS denied,
    round(100.0 * countIf(claim_status = 'denied') / count(), 2) AS denial_rate_pct
FROM claims
WHERE service_date >= today() - 90
GROUP BY payer_id, payer_type
HAVING claims_submitted > 50
ORDER BY denial_rate_pct DESC
LIMIT 20;
```

## Top Denial Reasons

```sql
SELECT
    denial_reason,
    count()                   AS denials,
    sum(billed_amount) / 1e6  AS billed_at_risk_m
FROM claims
WHERE claim_status = 'denied'
  AND service_date >= today() - 90
  AND denial_reason != ''
GROUP BY denial_reason
ORDER BY denials DESC
LIMIT 15;
```

## Cost Per Case by DRG

```sql
SELECT
    drg_code,
    count()                                     AS cases,
    round(avg(paid_amount) / 100, 2)            AS avg_paid,
    round(avg(los_days), 1)                     AS avg_los,
    round(avg(paid_amount) / 100 / avg(los_days), 2) AS cost_per_day
FROM claims
WHERE claim_status = 'paid'
  AND payer_type = 'medicare'
  AND service_date >= today() - 365
GROUP BY drg_code
ORDER BY cases DESC
LIMIT 20;
```

## Monthly Revenue Trend

```sql
SELECT
    toYYYYMM(service_date)              AS month,
    round(sum(paid_amount) / 1e6, 2)    AS revenue_m,
    count()                             AS paid_claims
FROM claims
WHERE claim_status = 'paid'
  AND service_date >= today() - 365
GROUP BY month
ORDER BY month;
```

## Accounts Receivable Aging

```sql
SELECT
    multiIf(
        dateDiff('day', billed_date, today()) <= 30,  '0-30',
        dateDiff('day', billed_date, today()) <= 60,  '31-60',
        dateDiff('day', billed_date, today()) <= 90,  '61-90',
        '90+'
    ) AS aging_bucket,
    count()                          AS claims,
    sum(billed_amount - paid_amount) / 1e6 AS outstanding_m
FROM claims
WHERE claim_status IN ('pending', 'denied')
GROUP BY aging_bucket
ORDER BY aging_bucket;
```

## Summary

ClickHouse makes healthcare cost analytics fast and flexible. Finance teams can drill into payer mix, denial reasons, cost-per-case by DRG, and AR aging across millions of claim lines in seconds. Columnar compression of repetitive payer and diagnosis codes reduces storage costs, while fast aggregation supports the iterative analysis common in revenue cycle management.
