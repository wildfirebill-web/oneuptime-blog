# How to Analyze Patient Readmission Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Healthcare, Readmission, Quality Metric, Care Management

Description: Identify 30-day readmission patterns in ClickHouse by analyzing diagnosis-specific rates, high-risk cohorts, and discharge disposition to reduce preventable readmissions.

---

Hospital readmission rates are a key quality metric and have direct financial implications under CMS value-based care programs. Identifying which patients, diagnoses, and discharge patterns drive readmissions is essential for care management teams.

## Encounter Table

```sql
CREATE TABLE encounters (
    encounter_id        UInt64,
    patient_id          UInt64,
    facility_id         UInt32,
    admit_date          Date,
    discharge_date      Date,
    los_days            UInt16,
    primary_icd10       LowCardinality(String),
    drg_code            LowCardinality(String),
    discharge_disp      LowCardinality(String),  -- 'home', 'snf', 'rehab', 'ama', 'expired'
    readmitted_30d      UInt8,
    encounter_type      LowCardinality(String),  -- 'inpatient', 'obs', 'ed'
    age_at_admit        UInt8,
    payer               LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (facility_id, patient_id, admit_date)
PARTITION BY toYear(admit_date);
```

## Overall 30-Day Readmission Rate

```sql
SELECT
    facility_id,
    count()                                      AS discharges,
    countIf(readmitted_30d = 1)                  AS readmissions,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_rate_pct
FROM encounters
WHERE encounter_type = 'inpatient'
  AND discharge_date >= today() - 365
  AND discharge_disp != 'expired'
GROUP BY facility_id
ORDER BY readmit_rate_pct DESC;
```

## Readmission Rate by Primary Diagnosis

```sql
SELECT
    primary_icd10,
    count()                                      AS discharges,
    countIf(readmitted_30d = 1)                  AS readmissions,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_rate_pct
FROM encounters
WHERE encounter_type = 'inpatient'
  AND discharge_date >= today() - 365
GROUP BY primary_icd10
HAVING discharges > 50
ORDER BY readmit_rate_pct DESC
LIMIT 20;
```

## Readmission by Discharge Disposition

```sql
SELECT
    discharge_disp,
    count()                                      AS discharges,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_pct
FROM encounters
WHERE discharge_date >= today() - 365
  AND encounter_type = 'inpatient'
GROUP BY discharge_disp
ORDER BY readmit_pct DESC;
```

## High-Risk Age Groups

```sql
SELECT
    multiIf(age_at_admit < 18, 'pediatric',
            age_at_admit < 65, 'adult',
            age_at_admit < 80, 'senior',
            'elderly') AS age_group,
    count()                                      AS discharges,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_pct
FROM encounters
WHERE encounter_type = 'inpatient'
  AND discharge_date >= today() - 365
GROUP BY age_group
ORDER BY readmit_pct DESC;
```

## Length of Stay vs Readmission

```sql
SELECT
    los_days,
    count()                                      AS discharges,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_pct
FROM encounters
WHERE encounter_type = 'inpatient'
  AND discharge_date >= today() - 365
  AND los_days <= 20
GROUP BY los_days
ORDER BY los_days;
```

## Monthly Readmission Trend

```sql
SELECT
    toYYYYMM(discharge_date)                     AS month,
    round(100.0 * countIf(readmitted_30d = 1) / count(), 2) AS readmit_pct
FROM encounters
WHERE encounter_type = 'inpatient'
  AND discharge_date >= today() - 365
GROUP BY month
ORDER BY month;
```

## Summary

ClickHouse enables care quality teams to rapidly analyze readmission patterns across facilities, diagnoses, and patient demographics. Conditional aggregation over large encounter datasets runs in seconds, making it practical to run readmission risk analyses daily - ensuring care managers have the most current data for proactive outreach to high-risk patients after discharge.
