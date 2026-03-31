# How to Use TS.REVRANGE in Redis to Query Time Series in Reverse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Time Series, Reverse Query, Latest Data

Description: Learn how to use TS.REVRANGE in Redis to query time series data in reverse chronological order, making it easy to get the most recent samples first.

---

## What Is TS.REVRANGE?

`TS.REVRANGE` queries a time series key within a time range and returns results in reverse chronological order (newest first). It is functionally identical to `TS.RANGE` except the results are returned from most recent to oldest.

This is useful when you want the latest N samples, since you can combine it with `COUNT` to efficiently get recent data without reading the entire series.

## Basic Syntax

```text
TS.REVRANGE key fromTimestamp toTimestamp
  [LATEST]
  [FILTER_BY_TS ts...]
  [FILTER_BY_VALUE min max]
  [COUNT count]
  [ALIGN align]
  [AGGREGATION aggregator bucketDuration [BUCKETTIMESTAMP bt] [EMPTY]]
```

The parameters are identical to `TS.RANGE`. The only difference is the sort order of results.

## Getting the Latest N Samples

```bash
# Add some samples
TS.ADD temperature:sensor1 1700000000000 22.5
TS.ADD temperature:sensor1 1700000001000 23.1
TS.ADD temperature:sensor1 1700000002000 21.8
TS.ADD temperature:sensor1 1700000003000 24.0
TS.ADD temperature:sensor1 1700000004000 22.2

# Get last 3 samples (newest first)
TS.REVRANGE temperature:sensor1 - + COUNT 3
```

```text
1) 1) (integer) 1700000004000
   2) "22.2"
2) 1) (integer) 1700000003000
   2) "24.0"
3) 1) (integer) 1700000002000
   2) "21.8"
```

## Comparison: RANGE vs REVRANGE

```bash
# TS.RANGE - oldest first
TS.RANGE temperature:sensor1 - + COUNT 3
# Returns: sample at T=0, T=1, T=2 (oldest 3)

# TS.REVRANGE - newest first
TS.REVRANGE temperature:sensor1 - + COUNT 3
# Returns: sample at T=4, T=3, T=2 (newest 3)
```

## Querying a Time Window in Reverse

```bash
# Get the last 5 minutes, newest first
# (timestamp range: 5 minutes ago to now)
TS.REVRANGE cpu:host1 1700000000000 1700000300000
```

## With Aggregation

When using aggregation with `REVRANGE`, buckets are still computed in forward time but returned in reverse order:

```bash
# Average per minute, returned newest-bucket first
TS.REVRANGE cpu:host1 - + AGGREGATION AVG 60000 COUNT 5
# Returns last 5 minute-averages, newest first
```

## Python Example - Recent Activity Feed

```python
import redis
import time
from datetime import datetime

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

# Get the 10 most recent temperature readings
samples = ts.revrange('temperature:sensor1', '-', '+', count=10)

print("Most recent temperature readings:")
for timestamp, value in samples:
    dt = datetime.fromtimestamp(timestamp / 1000)
    print(f"  {dt.strftime('%H:%M:%S')}: {float(value):.1f}C")
```

## Python Example - Detect Recent Anomalies

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

def find_recent_anomalies(key, threshold, lookback_count=100):
    """Find anomalous values in the most recent samples."""
    samples = ts.revrange(key, '-', '+', count=lookback_count)
    anomalies = []

    for timestamp, value in samples:
        if float(value) > threshold:
            anomalies.append((timestamp, float(value)))

    return anomalies

anomalies = find_recent_anomalies('cpu:host1', threshold=90.0)
if anomalies:
    print(f"Found {len(anomalies)} anomalies:")
    for ts_ms, val in anomalies[:5]:
        from datetime import datetime
        dt = datetime.fromtimestamp(ts_ms / 1000)
        print(f"  {dt}: {val:.1f}%")
else:
    print("No anomalies found")
```

## Getting Latest Value More Efficiently

For just the single latest value, `TS.GET` is more efficient. `TS.REVRANGE` is better when you need the latest N values:

```bash
# Single latest value - use TS.GET
TS.GET temperature:sensor1

# Latest 5 values - use TS.REVRANGE
TS.REVRANGE temperature:sensor1 - + COUNT 5

# Latest 5-minute average - use TS.REVRANGE with aggregation
TS.REVRANGE cpu:host1 - + AGGREGATION AVG 60000 COUNT 5
```

## Use Case: Sparkline Data

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

def get_sparkline_data(key, points=20, bucket_ms=60000):
    """Get data for a sparkline chart (most recent first)."""
    samples = ts.revrange(
        key, '-', '+',
        count=points,
        aggregation_type='avg',
        bucket_size_msec=bucket_ms
    )
    # Reverse to get chronological order for charting
    return list(reversed(samples))

data = get_sparkline_data('cpu:host1', points=20)
values = [float(v) for _, v in data]
print(f"Sparkline values: {values}")
```

## Summary

`TS.REVRANGE` provides reverse chronological access to time series data, making it the natural choice when you need the most recent samples first. It supports the same aggregation and filtering options as `TS.RANGE`, and combining it with `COUNT` efficiently retrieves recent windows without scanning old data. Common use cases include recent activity feeds, anomaly detection on recent values, and fetching sparkline data for dashboards.
