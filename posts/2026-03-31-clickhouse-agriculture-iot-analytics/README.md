# How to Use ClickHouse for Agriculture IoT Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Agriculture, IoT Analytics, Sensor Data, Precision Farming

Description: Use ClickHouse to ingest and analyze agriculture IoT sensor data including soil moisture, weather, crop yield, and irrigation metrics for precision farming.

---

## Agriculture IoT Data Characteristics

Precision farming generates continuous sensor streams from soil probes, weather stations, drones, and irrigation controllers. This data arrives at high frequency, is append-only, and requires time-window aggregations for actionable insights. ClickHouse's TimeSeries capabilities and columnar compression make it an ideal store.

## Sensor Data Schema

```sql
CREATE TABLE farm_sensor_readings
(
    reading_time DateTime,
    device_id UInt64,
    farm_id UInt32,
    field_id UInt32,
    sensor_type LowCardinality(String),
    latitude Float32,
    longitude Float32,
    soil_moisture_pct Float32,
    soil_temperature_c Float32,
    air_temperature_c Float32,
    humidity_pct Float32,
    rainfall_mm Float32,
    solar_radiation_wm2 Float32,
    battery_voltage Float32
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(reading_time)
ORDER BY (farm_id, field_id, sensor_type, reading_time)
TTL reading_time + INTERVAL 3 YEAR;
```

## Soil Moisture Dashboard

```sql
-- Current soil moisture status per field
SELECT
    field_id,
    any(farm_id) AS farm_id,
    avg(soil_moisture_pct) AS avg_moisture,
    min(soil_moisture_pct) AS min_moisture,
    max(soil_moisture_pct) AS max_moisture,
    max(reading_time) AS last_reading
FROM farm_sensor_readings
WHERE sensor_type = 'soil'
  AND reading_time >= now() - INTERVAL 2 HOUR
GROUP BY field_id
ORDER BY avg_moisture ASC;
```

## Irrigation Trigger Alerts

```sql
-- Fields needing irrigation (moisture below threshold)
SELECT
    field_id,
    farm_id,
    avg(soil_moisture_pct) AS avg_moisture_pct,
    max(reading_time) AS last_reading_time
FROM farm_sensor_readings
WHERE sensor_type = 'soil'
  AND reading_time >= now() - INTERVAL 1 HOUR
GROUP BY field_id, farm_id
HAVING avg_moisture_pct < 30.0  -- below 30% triggers irrigation
ORDER BY avg_moisture_pct;
```

## Growing Degree Days Calculation

Growing degree days (GDD) measure heat accumulation for crop development:

```sql
-- GDD accumulation since planting
SELECT
    farm_id,
    field_id,
    sum(
        greatest(0,
            (max_temp + min_temp) / 2.0 - 10.0  -- base temp 10C for corn
        )
    ) AS accumulated_gdd
FROM (
    SELECT
        farm_id,
        field_id,
        toDate(reading_time) AS reading_date,
        max(air_temperature_c) AS max_temp,
        min(air_temperature_c) AS min_temp
    FROM farm_sensor_readings
    WHERE reading_time >= '2026-04-01'
      AND sensor_type = 'weather'
    GROUP BY farm_id, field_id, reading_date
)
GROUP BY farm_id, field_id
ORDER BY accumulated_gdd DESC;
```

## Weather Pattern Analysis

```sql
-- Weekly rainfall and temperature trends
SELECT
    farm_id,
    toStartOfWeek(reading_time) AS week,
    sum(rainfall_mm) AS total_rainfall_mm,
    avg(air_temperature_c) AS avg_temp_c,
    max(air_temperature_c) AS max_temp_c,
    avg(humidity_pct) AS avg_humidity_pct
FROM farm_sensor_readings
WHERE sensor_type = 'weather'
  AND reading_time >= today() - 84
GROUP BY farm_id, week
ORDER BY farm_id, week;
```

## Device Health Monitoring

```sql
-- Detect sensors with low battery or missing readings
SELECT
    device_id,
    farm_id,
    max(reading_time) AS last_seen,
    avg(battery_voltage) AS avg_battery_v,
    dateDiff('hour', max(reading_time), now()) AS hours_since_last_reading
FROM farm_sensor_readings
WHERE reading_time >= now() - INTERVAL 48 HOUR
GROUP BY device_id, farm_id
HAVING hours_since_last_reading > 4 OR avg_battery_v < 3.3
ORDER BY hours_since_last_reading DESC;
```

## Summary

ClickHouse handles agriculture IoT analytics efficiently with its time-series storage model, fast aggregations, and high compression for numeric sensor data. Model your sensor readings with MergeTree partitioned by month and sorted by farm, field, and time to enable fast queries for soil monitoring, irrigation triggers, growing degree day calculations, and device health tracking.
