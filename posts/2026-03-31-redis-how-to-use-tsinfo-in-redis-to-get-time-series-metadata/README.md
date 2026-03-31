# How to Use TS.INFO in Redis to Get Time Series Metadata

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Time Series, Metadata, Monitoring

Description: Learn how to use TS.INFO in Redis to retrieve configuration, statistics, and compaction rules for a time series key for monitoring and debugging.

---

## What Is TS.INFO?

`TS.INFO` returns detailed metadata and statistics about a RedisTimeSeries key, including sample counts, memory usage, retention settings, labels, compaction rules, and encoding information. It is the primary command for inspecting the configuration and state of a time series key.

## Basic Syntax

```text
TS.INFO key [DEBUG]
```

Parameters:
- `key` - the time series key
- `DEBUG` - include additional internal statistics (chunk counts per bucket, etc.)

## Basic Usage

```bash
# Create and populate a time series
TS.CREATE cpu:host1 RETENTION 604800000 LABELS host host1 region us-east metric cpu
TS.ADD cpu:host1 * 72.5
TS.ADD cpu:host1 * 75.3
TS.ADD cpu:host1 * 68.9

# Get metadata
TS.INFO cpu:host1
```

Sample output:

```text
 1) totalSamples
 2) (integer) 3
 3) memoryUsage
 4) (integer) 4312
 5) firstTimestamp
 6) (integer) 1700000000000
 7) lastTimestamp
 8) (integer) 1700000002000
 9) retentionTime
10) (integer) 604800000
11) chunkCount
12) (integer) 1
13) chunkSize
14) (integer) 4096
15) chunkType
16) compressed
17) duplicatePolicy
18) (nil)
19) labels
20) 1) 1) "host"
       2) "host1"
    2) 1) "region"
       2) "us-east"
    3) 1) "metric"
       2) "cpu"
21) sourceKey
22) (nil)
23) rules
24) (empty array)
```

## Key Fields Explained

| Field | Description |
|---|---|
| totalSamples | Number of data points stored |
| memoryUsage | Memory used by this key in bytes |
| firstTimestamp | Timestamp of oldest sample |
| lastTimestamp | Timestamp of newest sample |
| retentionTime | Retention period in milliseconds (0 = forever) |
| chunkCount | Number of internal memory chunks |
| chunkSize | Size of each chunk in bytes |
| chunkType | compressed or uncompressed |
| duplicatePolicy | What happens on duplicate timestamps |
| labels | Key-value metadata labels |
| sourceKey | If this is a destination of a compaction rule, this shows the source |
| rules | Outgoing compaction rules (dest + bucket + aggregator) |

## Python Example

```python
import redis
from datetime import datetime

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

info = ts.info('cpu:host1')

print(f"Total samples: {info.total_samples}")
print(f"Memory usage: {info.memory_usage:,} bytes ({info.memory_usage / 1024:.1f} KB)")
print(f"Retention: {info.retention_msecs / 1000 / 3600:.1f} hours")

if info.first_timestamp:
    first_dt = datetime.fromtimestamp(info.first_timestamp / 1000)
    last_dt = datetime.fromtimestamp(info.last_timestamp / 1000)
    print(f"Data range: {first_dt} to {last_dt}")

print(f"Labels: {info.labels}")
print(f"Compaction rules: {info.rules}")
```

## Monitoring Multiple Series

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

def get_series_summary(key):
    """Get a concise summary of a time series."""
    try:
        info = ts.info(key)
        age_hours = 0
        if info.first_timestamp and info.last_timestamp:
            duration_ms = info.last_timestamp - info.first_timestamp
            age_hours = duration_ms / 1000 / 3600

        return {
            'key': key,
            'samples': info.total_samples,
            'memory_kb': info.memory_usage / 1024,
            'retention_hours': info.retention_msecs / 1000 / 3600 if info.retention_msecs else 'forever',
            'labels': info.labels,
            'rules': len(info.rules),
            'span_hours': age_hours,
        }
    except Exception as e:
        return {'key': key, 'error': str(e)}

keys = ['cpu:host1', 'memory:host1', 'disk:host1']
for key in keys:
    summary = get_series_summary(key)
    print(f"\n{summary['key']}:")
    for k, v in summary.items():
        if k != 'key':
            print(f"  {k}: {v}")
```

## Checking if Samples Are Stale

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

def check_freshness(key, max_age_seconds=120):
    """Check if the time series has recent data."""
    info = ts.info(key)

    if not info.last_timestamp:
        return False, "No data"

    age_seconds = (time.time() * 1000 - info.last_timestamp) / 1000

    if age_seconds > max_age_seconds:
        return False, f"Last sample is {age_seconds:.0f}s old"

    return True, f"Fresh (last sample {age_seconds:.0f}s ago)"

for key in ['cpu:host1', 'memory:host1']:
    is_fresh, reason = check_freshness(key)
    status = "OK" if is_fresh else "STALE"
    print(f"[{status}] {key}: {reason}")
```

## Using DEBUG Mode

```bash
TS.INFO cpu:host1 DEBUG
```

DEBUG mode adds a `keySelfName` field and `Chunks` array with detailed per-chunk statistics:

```text
...
Chunks:
1) 1) "startTimestamp"
   2) (integer) 1700000000000
   3) "endTimestamp"
   4) (integer) 1700000002000
   5) "samples"
   6) (integer) 3
   7) "size"
   8) (integer) 4096
   9) "bytesPerSample"
  10) "9.216"
```

## Summary

`TS.INFO` is the essential inspection command for RedisTimeSeries keys, providing a comprehensive view of a time series's configuration, data range, memory usage, labels, and compaction rules. Use it for routine monitoring of time series health, debugging unexpected behavior, verifying that compaction rules are set up correctly, and checking data freshness in alerting systems. The `DEBUG` option provides additional granularity for diagnosing storage efficiency issues.
