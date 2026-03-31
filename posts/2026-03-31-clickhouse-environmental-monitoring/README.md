# How to Use ClickHouse for Environmental Monitoring

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Environmental Monitoring, Air Quality, Sensor Data, Time Series

Description: Use ClickHouse to store and analyze environmental monitoring data including air quality, water quality, noise levels, and emissions tracking at city scale.

---

## Environmental Monitoring Data Characteristics

Environmental monitoring networks generate continuous time-series readings from thousands of sensors measuring air quality, water quality, noise levels, and meteorological conditions. This data is perfect for ClickHouse - it's high-volume, append-only, and requires time-window aggregations and anomaly detection queries.

## Sensor Readings Schema

```sql
CREATE TABLE env_sensor_readings
(
    reading_time DateTime,
    station_id UInt32,
    station_name LowCardinality(String),
    city LowCardinality(String),
    region LowCardinality(String),
    latitude Float32,
    longitude Float32,
    sensor_type LowCardinality(String),
    pm25 Float32,
    pm10 Float32,
    no2_ppb Float32,
    o3_ppb Float32,
    co_ppm Float32,
    so2_ppb Float32,
    temperature_c Float32,
    humidity_pct Float32,
    aqi UInt16
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(reading_time)
ORDER BY (city, station_id, reading_time)
TTL reading_time + INTERVAL 5 YEAR;
```

## Air Quality Index Dashboard

```sql
-- Current AQI status by city
SELECT
    city,
    avg(aqi) AS avg_aqi,
    max(aqi) AS max_aqi,
    multiIf(
        avg(aqi) <= 50,  'Good',
        avg(aqi) <= 100, 'Moderate',
        avg(aqi) <= 150, 'Unhealthy for Sensitive Groups',
        avg(aqi) <= 200, 'Unhealthy',
        avg(aqi) <= 300, 'Very Unhealthy',
        'Hazardous'
    ) AS aqi_category,
    count() AS active_stations
FROM env_sensor_readings
WHERE reading_time >= now() - INTERVAL 1 HOUR
GROUP BY city
ORDER BY avg_aqi DESC;
```

## Pollution Threshold Exceedance

```sql
-- Count hours exceeding WHO guidelines per month
SELECT
    station_id,
    city,
    toYYYYMM(reading_time) AS month,
    countIf(pm25 > 15) AS hours_pm25_exceeded,    -- WHO 24hr guideline: 15 ug/m3
    countIf(no2_ppb > 53) AS hours_no2_exceeded,  -- EPA standard
    countIf(o3_ppb > 70) AS hours_o3_exceeded
FROM env_sensor_readings
WHERE reading_time >= today() - 365
GROUP BY station_id, city, month
ORDER BY hours_pm25_exceeded DESC;
```

## Trend Analysis with Moving Averages

```sql
-- 7-day rolling average PM2.5 by city
SELECT
    city,
    toDate(reading_time) AS date,
    avg(pm25) AS daily_avg_pm25,
    avg(avg(pm25)) OVER (
        PARTITION BY city
        ORDER BY toDate(reading_time)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day_avg
FROM env_sensor_readings
WHERE reading_time >= today() - 60
GROUP BY city, date
ORDER BY city, date;
```

## Anomaly Detection

```sql
-- Stations with sudden PM2.5 spikes (3 standard deviations above mean)
WITH baseline AS (
    SELECT
        station_id,
        avg(pm25) AS mean_pm25,
        stddevPop(pm25) AS std_pm25
    FROM env_sensor_readings
    WHERE reading_time BETWEEN today() - 30 AND today() - 1
    GROUP BY station_id
)
SELECT
    r.station_id,
    r.city,
    r.reading_time,
    r.pm25,
    b.mean_pm25,
    b.std_pm25,
    (r.pm25 - b.mean_pm25) / b.std_pm25 AS z_score
FROM env_sensor_readings AS r
JOIN baseline AS b ON r.station_id = b.station_id
WHERE r.reading_time >= now() - INTERVAL 1 HOUR
  AND (r.pm25 - b.mean_pm25) / b.std_pm25 > 3
ORDER BY z_score DESC;
```

## Summary

ClickHouse is an excellent fit for environmental monitoring because sensor data is append-only, arrives continuously, and requires fast time-series aggregations. Use MergeTree partitioned by month and sorted by city, station, and time to power AQI dashboards, exceedance tracking, trend analysis with moving averages, and statistical anomaly detection across city-scale sensor networks.
