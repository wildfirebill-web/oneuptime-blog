# How to Track Environmental Conditions in Cold Chain with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cold Chain, IoT, Temperature, Monitoring, Compliance

Description: Monitor cold chain environmental conditions in ClickHouse - track temperature and humidity excursions, compliance windows, and alert thresholds for perishable goods.

---

Cold chain logistics requires continuous monitoring of temperature and humidity to ensure product integrity. ClickHouse stores and analyzes high-frequency sensor data from thousands of devices across refrigerated trucks, warehouses, and containers.

## Sensor Readings Table

```sql
CREATE TABLE cold_chain_readings (
    device_id    LowCardinality(String),
    shipment_id  String,
    asset_type   LowCardinality(String),  -- truck, warehouse, container
    asset_id     LowCardinality(String),
    temperature  Float32,  -- celsius
    humidity     Float32,  -- percent
    door_open    UInt8,
    battery_pct  Float32,
    recorded_at  DateTime64(3)
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (device_id, recorded_at);
```

## Temperature Excursion Detection

Identify devices with readings outside acceptable ranges:

```sql
SELECT
    device_id,
    shipment_id,
    asset_type,
    min(temperature) AS min_temp,
    max(temperature) AS max_temp,
    countIf(temperature < -22 OR temperature > -18) AS excursion_readings,
    count() AS total_readings,
    round(countIf(temperature < -22 OR temperature > -18) / count() * 100, 2) AS excursion_pct
FROM cold_chain_readings
WHERE recorded_at >= today() - 7
  AND asset_type = 'truck'
GROUP BY device_id, shipment_id, asset_type
HAVING excursion_readings > 0
ORDER BY excursion_pct DESC;
```

## Excursion Duration Calculation

Measure how long each device stayed out of range:

```sql
SELECT
    device_id,
    shipment_id,
    recorded_at,
    temperature,
    temperature < -22 OR temperature > -18 AS is_excursion,
    runningDifference(toUnixTimestamp(recorded_at)) AS seconds_since_last
FROM cold_chain_readings
WHERE recorded_at >= today() - 1
  AND device_id = 'TRUCK_001'
ORDER BY device_id, recorded_at;
```

## Mean Kinetic Temperature (MKT)

Compute MKT, a pharmaceutical compliance metric for temperature exposure:

```sql
SELECT
    shipment_id,
    device_id,
    count() AS readings,
    avg(temperature) AS arithmetic_mean_temp,
    -- MKT approximation: simplified for illustration
    log(sum(exp(temperature / 8.314)) / count()) * 8.314 AS approx_mkt
FROM cold_chain_readings
WHERE recorded_at >= today() - 7
GROUP BY shipment_id, device_id
ORDER BY approx_mkt DESC;
```

## Door Open Events

Frequent door openings cause temperature spikes - track them:

```sql
SELECT
    asset_id,
    toDate(recorded_at) AS day,
    countIf(door_open = 1) AS door_open_events,
    avg(temperature) AS avg_temp,
    maxIf(temperature, door_open = 1) AS max_temp_during_open
FROM cold_chain_readings
WHERE recorded_at >= today() - 7
GROUP BY asset_id, day
ORDER BY door_open_events DESC;
```

## Compliance Summary by Shipment

Summarize overall cold chain compliance per shipment:

```sql
SELECT
    shipment_id,
    min(recorded_at) AS monitoring_start,
    max(recorded_at) AS monitoring_end,
    dateDiff('hour', min(recorded_at), max(recorded_at)) AS monitored_hours,
    min(temperature) AS min_temp,
    max(temperature) AS max_temp,
    avg(temperature) AS avg_temp,
    countIf(temperature BETWEEN -22 AND -18) AS compliant_readings,
    count() AS total_readings,
    round(countIf(temperature BETWEEN -22 AND -18) / count() * 100, 2) AS compliance_pct
FROM cold_chain_readings
WHERE recorded_at >= today() - 30
GROUP BY shipment_id
ORDER BY compliance_pct;
```

## Summary

ClickHouse excels at cold chain monitoring with its ability to ingest high-frequency sensor data and quickly identify temperature excursions, compute compliance metrics, and flag shipments at risk. Partitioning by day keeps recent data queries fast while long-term compliance reports scan historical partitions efficiently.
