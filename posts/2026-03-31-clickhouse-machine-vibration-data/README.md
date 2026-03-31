# How to Analyze Machine Vibration Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Vibration, Condition Monitoring, Manufacturing, FFT, Analytics

Description: Analyze machine vibration data in ClickHouse to detect bearing faults, imbalance, and resonance patterns using statistical features and trend analysis.

---

Vibration analysis is one of the most effective condition monitoring techniques for rotating equipment. ClickHouse stores high-frequency vibration measurements and computes statistical features that feed predictive maintenance models.

## Vibration Measurements Table

```sql
CREATE TABLE vibration_data (
    machine_id   LowCardinality(String),
    sensor_pos   LowCardinality(String),  -- drive_end, non_drive_end, radial, axial
    axis         LowCardinality(String),  -- x, y, z
    rms_mm_s     Float32,   -- RMS velocity mm/s
    peak_mm_s    Float32,   -- peak velocity
    crest_factor Float32,   -- peak/RMS ratio, indicates impulsiveness
    kurtosis     Float32,   -- statistical kurtosis, sensitive to impacts
    temperature_c Float32,
    speed_rpm    Float32,
    recorded_at  DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (machine_id, sensor_pos, recorded_at);
```

## RMS Velocity Trend

Track vibration severity over time - ISO 10816 severity zones:

```sql
SELECT
    machine_id,
    sensor_pos,
    toStartOfHour(recorded_at) AS hour,
    avg(rms_mm_s) AS avg_rms,
    max(rms_mm_s) AS max_rms,
    multiIf(
        avg(rms_mm_s) < 2.3, 'Zone A - New',
        avg(rms_mm_s) < 4.5, 'Zone B - Good',
        avg(rms_mm_s) < 7.1, 'Zone C - Alarm',
        'Zone D - Danger'
    ) AS iso_zone
FROM vibration_data
WHERE recorded_at >= today() - 30
GROUP BY machine_id, sensor_pos, hour
ORDER BY machine_id, sensor_pos, hour;
```

## Kurtosis Spike Detection

High kurtosis indicates impulsive events like bearing spalling:

```sql
SELECT
    machine_id,
    sensor_pos,
    recorded_at,
    kurtosis,
    rms_mm_s,
    avg(kurtosis) OVER (
        PARTITION BY machine_id, sensor_pos
        ORDER BY recorded_at
        ROWS BETWEEN 143 PRECEDING AND CURRENT ROW
    ) AS rolling_24h_avg_kurtosis
FROM vibration_data
WHERE recorded_at >= today() - 7
HAVING kurtosis > 3 * rolling_24h_avg_kurtosis
   AND kurtosis > 6
ORDER BY kurtosis DESC;
```

## Crest Factor Trend

Rising crest factor indicates early bearing damage before RMS increases:

```sql
SELECT
    machine_id,
    sensor_pos,
    toStartOfDay(recorded_at) AS day,
    avg(crest_factor) AS avg_crest_factor,
    avg(crest_factor) - lag(avg(crest_factor), 7) OVER (
        PARTITION BY machine_id, sensor_pos ORDER BY toStartOfDay(recorded_at)
    ) AS crest_change_7d
FROM vibration_data
WHERE recorded_at >= today() - 60
GROUP BY machine_id, sensor_pos, day
ORDER BY machine_id, sensor_pos, day;
```

## Speed-Corrected Comparison

Compare vibration severity at similar speeds to filter out speed effects:

```sql
SELECT
    machine_id,
    round(speed_rpm / 100) * 100 AS speed_bucket,
    avg(rms_mm_s) AS avg_rms,
    avg(kurtosis) AS avg_kurtosis,
    count() AS measurements
FROM vibration_data
WHERE recorded_at >= today() - 30
  AND speed_rpm > 0
GROUP BY machine_id, speed_bucket
ORDER BY machine_id, speed_bucket;
```

## Fleet Comparison

Benchmark vibration severity across identical machines:

```sql
SELECT
    machine_id,
    avg(rms_mm_s) AS avg_rms_7d,
    avg(avg(rms_mm_s)) OVER () AS fleet_avg_rms,
    avg(rms_mm_s) / nullIf(avg(avg(rms_mm_s)) OVER (), 0) AS relative_severity
FROM vibration_data
WHERE recorded_at >= today() - 7
GROUP BY machine_id
ORDER BY relative_severity DESC;
```

## Summary

ClickHouse stores and analyzes machine vibration data effectively - RMS trends, kurtosis spikes, crest factor trajectories, and fleet benchmarking queries are all fast with columnar storage. Statistical vibration features computed in ClickHouse feed directly into predictive maintenance models and condition monitoring dashboards.
