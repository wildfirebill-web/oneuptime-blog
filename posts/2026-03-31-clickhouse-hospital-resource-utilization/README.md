# How to Analyze Hospital Resource Utilization with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Healthcare, Hospital Operations, Capacity Planning, Bed Management

Description: Analyze hospital bed occupancy, OR utilization, staff ratios, and equipment availability in ClickHouse to improve operational efficiency.

---

Hospital administrators need real-time visibility into resource utilization to prevent bottlenecks, reduce costs, and improve patient flow. ClickHouse can aggregate data from bed management systems, OR scheduling tools, and staffing platforms to produce actionable operational dashboards.

## Resource Utilization Events Table

```sql
CREATE TABLE hospital_utilization (
    event_id        UUID,
    facility_id     UInt32,
    unit            LowCardinality(String),   -- 'ICU', 'ED', 'OR', 'Med/Surg'
    resource_type   LowCardinality(String),   -- 'bed', 'or_suite', 'ventilator', 'nurse'
    resource_id     UInt32,
    recorded_at     DateTime,
    is_occupied     UInt8,
    patient_acuity  LowCardinality(String),   -- 'low', 'medium', 'high', 'critical'
    census          UInt16,   -- current count in unit
    capacity        UInt16,
    staff_count     UInt16
) ENGINE = MergeTree()
ORDER BY (facility_id, unit, resource_type, recorded_at)
PARTITION BY toYYYYMM(recorded_at);
```

## Average Occupancy Rate by Unit

```sql
SELECT
    unit,
    round(100.0 * avg(census) / avg(capacity), 1)  AS avg_occupancy_pct,
    round(max(census) / max(capacity) * 100, 1)    AS peak_occupancy_pct
FROM hospital_utilization
WHERE resource_type = 'bed'
  AND recorded_at >= today() - 30
GROUP BY unit
ORDER BY avg_occupancy_pct DESC;
```

## ICU Capacity Alerts

```sql
SELECT
    facility_id,
    unit,
    recorded_at,
    census,
    capacity,
    round(100.0 * census / capacity, 1) AS occupancy_pct
FROM hospital_utilization
WHERE resource_type = 'bed'
  AND unit = 'ICU'
  AND recorded_at >= now() - INTERVAL 4 HOUR
  AND census >= capacity * 0.9
ORDER BY recorded_at DESC
LIMIT 20;
```

## OR Suite Utilization by Hour

```sql
SELECT
    toHour(recorded_at) AS hour,
    round(100.0 * avg(is_occupied), 1) AS avg_utilization_pct
FROM hospital_utilization
WHERE resource_type = 'or_suite'
  AND recorded_at >= today() - 30
GROUP BY hour
ORDER BY hour;
```

## Staff-to-Patient Ratio Monitoring

```sql
SELECT
    unit,
    toDate(recorded_at)              AS day,
    round(avg(census), 1)            AS avg_census,
    round(avg(staff_count), 1)       AS avg_staff,
    round(avg(census) / nullIf(avg(staff_count), 0), 2) AS patient_staff_ratio
FROM hospital_utilization
WHERE resource_type = 'nurse'
  AND recorded_at >= today() - 14
GROUP BY unit, day
ORDER BY day, unit;
```

## Equipment Availability

```sql
SELECT
    resource_type,
    count()                         AS total_units,
    countIf(is_occupied = 0)        AS available,
    round(100.0 * countIf(is_occupied = 1) / count(), 1) AS utilization_pct
FROM hospital_utilization
WHERE resource_type IN ('ventilator', 'monitor')
  AND recorded_at >= now() - INTERVAL 30 MINUTE
GROUP BY resource_type;
```

## Hourly Census Trend (Last 7 Days)

```sql
SELECT
    toStartOfHour(recorded_at) AS hour,
    unit,
    round(avg(census), 0)      AS avg_census
FROM hospital_utilization
WHERE resource_type = 'bed'
  AND recorded_at >= today() - 7
GROUP BY hour, unit
ORDER BY hour, unit;
```

## Summary

ClickHouse provides hospital operations teams with fast analytical access to resource utilization data at the unit and facility level. Conditional aggregation, rolling averages, and hourly heatmaps surface bed shortages, understaffing events, and equipment bottlenecks - enabling proactive capacity management rather than reactive crisis response.
