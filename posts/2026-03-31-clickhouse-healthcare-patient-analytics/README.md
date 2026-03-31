# How to Build a Healthcare Patient Analytics System with ClickHouse

Author: [OneUptime](https://www.github.com/oneuptime)

Tags: ClickHouse, Analytics, Healthcare, Patient Data, Time-Series

Description: Design a HIPAA-aware ClickHouse schema to analyze patient vitals, admissions, lab results, and care outcomes at population scale with fast query performance.

## Introduction

Healthcare analytics demands a system that can handle high-frequency vitals from ICU monitors, millions of lab results, decades of admission records, and complex cohort queries - all while maintaining strict access controls. ClickHouse fits this workload because of its ability to aggregate large time-series datasets quickly and its support for role-based access control.

This guide covers schema design for patient vitals, admissions, lab results, and outcomes, along with the analytical queries that power population health dashboards.

> Note: Any real deployment must comply with applicable regulations (HIPAA, GDPR, etc.). Encrypt data at rest, apply column-level access controls, and audit all queries. Patient identifiers shown here use anonymized IDs.

## Schema Design

### Patient Vitals

```sql
CREATE TABLE patient_vitals
(
    patient_id      String,
    ward            LowCardinality(String),
    recorded_at     DateTime64(3),
    device_id       LowCardinality(String),
    heart_rate      Nullable(UInt16),
    systolic_bp     Nullable(UInt16),
    diastolic_bp    Nullable(UInt16),
    spo2_pct        Nullable(Float32),
    temperature_c   Nullable(Float32),
    respiratory_rate Nullable(UInt16),
    glucose_mmol    Nullable(Float32)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (ward, patient_id, recorded_at)
TTL recorded_at + INTERVAL 10 YEAR;
```

### Admission and Discharge Records

```sql
CREATE TABLE patient_admissions
(
    admission_id     String,
    patient_id       String,
    admitted_at      DateTime,
    discharged_at    Nullable(DateTime),
    department       LowCardinality(String),
    diagnosis_codes  Array(String),
    drg_code         LowCardinality(String),
    attending_id     String,
    facility_id      LowCardinality(String),
    admission_type   LowCardinality(String),  -- Emergency, Elective, Transfer
    discharge_type   LowCardinality(String)   -- Home, Transfer, Deceased, AMA
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(admitted_at)
ORDER BY (facility_id, department, admitted_at);
```

### Lab Results

```sql
CREATE TABLE lab_results
(
    result_id       UUID DEFAULT generateUUIDv4(),
    patient_id      String,
    ordered_at      DateTime,
    resulted_at     DateTime,
    test_code       LowCardinality(String),
    test_name       LowCardinality(String),
    value_numeric   Nullable(Float64),
    value_text      String,
    unit            LowCardinality(String),
    ref_range_low   Nullable(Float64),
    ref_range_high  Nullable(Float64),
    is_abnormal     UInt8,
    facility_id     LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ordered_at)
ORDER BY (patient_id, test_code, ordered_at);
```

### Medication Administration

```sql
CREATE TABLE medication_administrations
(
    admin_id        UUID DEFAULT generateUUIDv4(),
    patient_id      String,
    administered_at DateTime,
    drug_code       LowCardinality(String),
    drug_name       LowCardinality(String),
    dose_mg         Float32,
    route           LowCardinality(String),  -- IV, Oral, IM, SubQ
    nurse_id        String,
    facility_id     LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(administered_at)
ORDER BY (patient_id, administered_at);
```

## Inserting Sample Data

```sql
INSERT INTO patient_vitals
    (patient_id, ward, recorded_at, device_id, heart_rate, systolic_bp, diastolic_bp, spo2_pct, temperature_c, respiratory_rate)
VALUES
    ('PT-0001', 'ICU-A', now64(), 'MON-101', 88,  128, 82, 97.5, 37.1, 18),
    ('PT-0001', 'ICU-A', now64() - 300000, 'MON-101', 92, 132, 85, 96.8, 37.3, 20),
    ('PT-0002', 'WARD-3', now64(), 'MON-204', 72, 118, 76, 98.2, 36.9, 16);
```

## Average Vitals by Ward

```sql
SELECT
    ward,
    round(avg(heart_rate), 1)        AS avg_hr,
    round(avg(systolic_bp), 1)       AS avg_systolic,
    round(avg(spo2_pct), 2)          AS avg_spo2,
    round(avg(temperature_c), 2)     AS avg_temp_c
FROM patient_vitals
WHERE recorded_at >= now() - INTERVAL 24 HOUR
GROUP BY ward
ORDER BY ward;
```

## Detecting Patients with Critical SpO2

```sql
SELECT
    patient_id,
    ward,
    min(spo2_pct)     AS lowest_spo2,
    max(recorded_at)  AS last_reading
FROM patient_vitals
WHERE recorded_at >= now() - INTERVAL 1 HOUR
  AND spo2_pct < 90
GROUP BY patient_id, ward
ORDER BY lowest_spo2;
```

## Length of Stay Analysis

```sql
SELECT
    department,
    admission_type,
    count()                                                      AS admissions,
    avg(dateDiff('hour', admitted_at, coalesce(discharged_at, now()))) AS avg_los_hours,
    median(dateDiff('hour', admitted_at, coalesce(discharged_at, now()))) AS median_los_hours,
    countIf(discharge_type = 'Deceased') / count() * 100        AS mortality_rate_pct
FROM patient_admissions
WHERE admitted_at >= now() - INTERVAL 365 DAY
GROUP BY department, admission_type
ORDER BY avg_los_hours DESC;
```

## Readmission Rate (30-Day)

Identifying patients who return within 30 days after discharge is a key quality metric:

```sql
SELECT
    a1.patient_id,
    a1.discharge_type,
    a1.department,
    min(a2.admitted_at) AS readmission_date,
    dateDiff('day', a1.discharged_at, min(a2.admitted_at)) AS days_to_readmission
FROM patient_admissions AS a1
JOIN patient_admissions AS a2
    ON a1.patient_id = a2.patient_id
    AND a2.admitted_at > a1.discharged_at
    AND a2.admitted_at <= a1.discharged_at + INTERVAL 30 DAY
WHERE a1.discharged_at >= now() - INTERVAL 180 DAY
  AND a1.discharge_type = 'Home'
GROUP BY a1.patient_id, a1.discharge_type, a1.department, a1.discharged_at
ORDER BY days_to_readmission;
```

## Abnormal Lab Result Rate by Test

```sql
SELECT
    test_code,
    test_name,
    count()                              AS total_results,
    sum(is_abnormal)                     AS abnormal_count,
    round(avg(is_abnormal) * 100, 2)    AS abnormal_rate_pct,
    avg(value_numeric)                   AS avg_value,
    any(unit)                            AS unit
FROM lab_results
WHERE resulted_at >= now() - INTERVAL 30 DAY
GROUP BY test_code, test_name
HAVING total_results > 50
ORDER BY abnormal_rate_pct DESC
LIMIT 20;
```

## Turnaround Time for Labs

```sql
SELECT
    test_code,
    test_name,
    count()                                                    AS result_count,
    avg(dateDiff('minute', ordered_at, resulted_at))           AS avg_tat_min,
    quantile(0.95)(dateDiff('minute', ordered_at, resulted_at)) AS p95_tat_min
FROM lab_results
WHERE ordered_at >= now() - INTERVAL 7 DAY
GROUP BY test_code, test_name
ORDER BY avg_tat_min DESC
LIMIT 20;
```

## Cohort Analysis: Diabetic Patients

```sql
WITH diabetic_patients AS (
    SELECT DISTINCT patient_id
    FROM patient_admissions
    WHERE has(diagnosis_codes, 'E11')   -- ICD-10: Type 2 diabetes
)
SELECT
    toDate(resulted_at)                              AS day,
    avg(value_numeric)                               AS avg_hba1c,
    countIf(value_numeric > 8.0) / count() * 100    AS pct_poorly_controlled
FROM lab_results
WHERE patient_id IN (SELECT patient_id FROM diabetic_patients)
  AND test_code = 'HBA1C'
  AND resulted_at >= now() - INTERVAL 365 DAY
GROUP BY day
ORDER BY day;
```

## Medication Frequency by Drug

```sql
SELECT
    drug_name,
    route,
    count()               AS administrations,
    count(DISTINCT patient_id) AS unique_patients,
    avg(dose_mg)          AS avg_dose_mg
FROM medication_administrations
WHERE administered_at >= toStartOfMonth(now())
GROUP BY drug_name, route
ORDER BY administrations DESC
LIMIT 30;
```

## Hourly ICU Alert Summary Materialized View

```sql
CREATE MATERIALIZED VIEW icu_hourly_vitals_mv
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (ward, hour)
AS
SELECT
    ward,
    toStartOfHour(recorded_at)           AS hour,
    avgState(heart_rate)                 AS avg_hr_state,
    avgState(spo2_pct)                   AS avg_spo2_state,
    countIfState(spo2_pct < 90)          AS critical_spo2_count_state,
    countIfState(heart_rate > 120)       AS tachycardia_count_state,
    countIfState(systolic_bp > 180)      AS hypertension_count_state
FROM patient_vitals
GROUP BY ward, hour;
```

## Conclusion

ClickHouse gives healthcare organizations the ability to query patient datasets at population scale without sacrificing response time. The columnar storage model makes aggregations over millions of vitals, lab results, and admissions fast enough for interactive dashboards. Combined with proper RBAC, column-level encryption, and audit logging, ClickHouse can serve as the analytics layer in a compliant healthcare data platform.

**Related Reading:**

- [How to Detect Anomalies in Time-Series Data with ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-anomaly-detection-time-series/view)
- [What Is TTL in ClickHouse and How Data Lifecycle Works](https://oneuptime.com/blog/post/2026-03-31-clickhouse-ttl-data-lifecycle/view)
- [What Is ClickHouse MergeTree and How It Works](https://oneuptime.com/blog/post/2026-03-31-clickhouse-mergetree-explained/view)
