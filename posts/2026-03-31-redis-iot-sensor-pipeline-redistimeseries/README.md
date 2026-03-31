# How to Build an IoT Sensor Data Pipeline with RedisTimeSeries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, IoT, Time Series

Description: Build a real-time IoT sensor ingestion pipeline with RedisTimeSeries for storing, querying, and downsampling high-frequency sensor readings.

---

IoT deployments generate high-frequency sensor data that needs to be stored efficiently, queried quickly, and automatically downsampled for long-term retention. RedisTimeSeries is purpose-built for this pattern.

## Installation

```bash
pip install redis
```

Ensure your Redis server has the RedisTimeSeries module loaded:

```bash
redis-server --loadmodule /path/to/redistimeseries.so
```

Or use the Redis Stack Docker image:

```bash
docker run -p 6379:6379 redis/redis-stack-server:latest
```

## Creating Time Series Keys

Create a time series for each sensor metric with labels:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_sensor_series(device_id: str, metric: str,
                          retention_ms: int = 86400000):
    key = f"sensor:{device_id}:{metric}"
    try:
        r.execute_command(
            'TS.CREATE', key,
            'RETENTION', retention_ms,
            'LABELS',
            'device_id', device_id,
            'metric', metric,
            'type', 'sensor'
        )
    except Exception:
        pass  # already exists
    return key

# Create series for temperature and humidity sensors
create_sensor_series("device-001", "temperature", 7 * 86400000)
create_sensor_series("device-001", "humidity", 7 * 86400000)
create_sensor_series("device-002", "temperature", 7 * 86400000)
```

## Setting Up Compaction Rules

Automatically downsample raw data into minute and hour averages:

```python
def setup_compaction(device_id: str, metric: str):
    raw_key = f"sensor:{device_id}:{metric}"
    min_key = f"sensor:{device_id}:{metric}:1min"
    hour_key = f"sensor:{device_id}:{metric}:1hour"

    # 1-minute averages, retain 30 days
    try:
        r.execute_command(
            'TS.CREATE', min_key,
            'RETENTION', 30 * 86400000,
            'LABELS', 'device_id', device_id,
            'metric', metric, 'resolution', '1min'
        )
        r.execute_command(
            'TS.CREATERULE', raw_key, min_key,
            'AGGREGATION', 'avg', 60000
        )
    except Exception:
        pass

    # 1-hour averages, retain 1 year
    try:
        r.execute_command(
            'TS.CREATE', hour_key,
            'RETENTION', 365 * 86400000,
            'LABELS', 'device_id', device_id,
            'metric', metric, 'resolution', '1hour'
        )
        r.execute_command(
            'TS.CREATERULE', raw_key, hour_key,
            'AGGREGATION', 'avg', 3600000
        )
    except Exception:
        pass

setup_compaction("device-001", "temperature")
```

## Ingesting Sensor Readings

Add readings as they arrive - use MADD for batch ingestion:

```python
def ingest_reading(device_id: str, metric: str, value: float,
                   timestamp_ms: int = None):
    key = f"sensor:{device_id}:{metric}"
    ts = timestamp_ms or int(time.time() * 1000)
    r.execute_command('TS.ADD', key, ts, value)

def ingest_batch(readings: list):
    # readings = [(device_id, metric, timestamp_ms, value), ...]
    args = []
    for device_id, metric, ts, value in readings:
        key = f"sensor:{device_id}:{metric}"
        args.extend([key, ts, value])
    r.execute_command('TS.MADD', *args)

# Simulate sensor data
import random
for i in range(10):
    ingest_reading("device-001", "temperature",
                   round(22.5 + random.uniform(-2, 2), 2))
    ingest_reading("device-001", "humidity",
                   round(55 + random.uniform(-5, 5), 2))
```

## Querying Recent Data

```python
def get_recent_readings(device_id: str, metric: str,
                         minutes: int = 60):
    key = f"sensor:{device_id}:{metric}"
    now_ms = int(time.time() * 1000)
    from_ms = now_ms - minutes * 60 * 1000

    result = r.execute_command('TS.RANGE', key, from_ms, now_ms)
    return [{"ts": ts, "value": float(val)} for ts, val in result]

recent = get_recent_readings("device-001", "temperature", 30)
```

## Querying Across All Devices

Use TS.MRANGE to query multiple sensors by label:

```bash
TS.MRANGE - + FILTER metric=temperature type=sensor
```

```python
def get_all_device_readings(metric: str):
    result = r.execute_command(
        'TS.MRANGE', '-', '+',
        'FILTER', f'metric={metric}'
    )
    return result
```

## Summary

RedisTimeSeries simplifies IoT data pipelines by handling ingestion, automatic downsampling via compaction rules, and fast range queries in a single module. Store raw high-frequency readings with a short retention window, and use compaction rules to automatically create minute and hourly aggregates for dashboards and long-term analysis.
