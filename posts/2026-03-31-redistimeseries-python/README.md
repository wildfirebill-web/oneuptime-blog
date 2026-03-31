# How to Use RedisTimeSeries in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, RedisTimeSeries, Time Series, Monitoring

Description: Store and query time-series data in Redis using the RedisTimeSeries module with redis-py, covering ingestion, downsampling rules, and range queries.

---

RedisTimeSeries is a Redis module that adds native time-series storage with automatic downsampling, retention policies, and aggregation queries. It is ideal for metrics, IoT sensor data, and financial tick data.

## Prerequisites

```bash
docker run -d -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Creating a Time Series

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Create a time series with 30-day retention and labels
r.ts().create(
    "sensor:cpu",
    retention_msecs=30 * 24 * 60 * 60 * 1000,  # 30 days in ms
    labels={"host": "web-01", "metric": "cpu_usage"},
)
print("Time series created")
```

## Adding Data Points

```python
import time

# Add a single data point (timestamp in milliseconds, or use * for server time)
ts_ms = int(time.time() * 1000)
r.ts().add("sensor:cpu", ts_ms, 72.5)

# Auto-timestamp with "*"
r.ts().add("sensor:cpu", "*", 74.1)

# Bulk add multiple samples
samples = [
    ("sensor:cpu", ts_ms + 1000, 75.0),
    ("sensor:memory", ts_ms + 1000, 68.4),
]
r.ts().madd(samples)
```

## Querying Data

```python
# Get the latest value
latest = r.ts().get("sensor:cpu")
print(f"Timestamp: {latest[0]}  Value: {latest[1]}")

# Range query - last 5 minutes
now_ms = int(time.time() * 1000)
five_min_ago = now_ms - (5 * 60 * 1000)

data_points = r.ts().range("sensor:cpu", five_min_ago, now_ms)
for ts, value in data_points:
    print(f"{ts}: {value}")

# Reverse range (newest first)
data_points = r.ts().revrange("sensor:cpu", five_min_ago, now_ms, count=100)
```

## Aggregation in Range Queries

```python
from redis.commands.timeseries import TimeSeries

# Average CPU per minute over last hour
one_hour_ago = now_ms - (60 * 60 * 1000)
avg_per_minute = r.ts().range(
    "sensor:cpu",
    one_hour_ago,
    now_ms,
    aggregation_type="avg",
    bucket_size_msec=60 * 1000,
)
for ts, avg in avg_per_minute:
    print(f"{ts}: avg={avg:.2f}%")
```

Supported aggregation types: `avg`, `sum`, `min`, `max`, `count`, `first`, `last`, `std.p`, `std.s`.

## Automatic Downsampling with Compaction Rules

Create a compaction rule to automatically compute hourly averages from raw minutely data:

```python
# Create destination series for hourly rollups
r.ts().create(
    "sensor:cpu:hourly",
    retention_msecs=365 * 24 * 60 * 60 * 1000,  # 1 year
    labels={"host": "web-01", "metric": "cpu_usage", "resolution": "1h"},
)

# Create compaction rule
r.ts().createrule(
    source_key="sensor:cpu",
    dest_key="sensor:cpu:hourly",
    aggregation_type="avg",
    bucket_size_msec=60 * 60 * 1000,  # 1 hour buckets
)
```

## Multi-Series Query with MRANGE

```python
# Query all CPU sensors by label filter
results = r.ts().mrange(
    one_hour_ago,
    now_ms,
    filters=["metric=cpu_usage"],
    aggregation_type="avg",
    bucket_size_msec=60 * 1000,
)
for key, labels, data in results:
    print(f"{key}: {len(data)} data points")
```

## Summary

RedisTimeSeries provides efficient time-series storage with retention policies, label-based filtering, and compaction rules for automatic downsampling. Use redis-py's `ts()` client for ingestion with `ts().add()` and `ts().madd()`, and range queries with aggregation for dashboarding and anomaly detection directly from Python.
