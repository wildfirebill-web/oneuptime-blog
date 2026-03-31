# How to Use TS.MADD in Redis to Add Multiple Time Series Samples

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Time Series, Batch Ingestion, Metric

Description: Learn how to use TS.MADD in Redis to atomically add multiple timestamped samples to different time series keys in a single command.

---

## What Is TS.MADD?

`TS.MADD` adds multiple samples across multiple time series keys in a single command. It is the batch variant of `TS.ADD`, allowing you to insert data into many keys at once, which is more efficient than making multiple individual `TS.ADD` calls.

All samples in a `TS.MADD` call are processed atomically (at the Redis command level), though each key gets its own timestamp.

## Basic Syntax

```text
TS.MADD {key timestamp value}...
```

Each triplet `key timestamp value` specifies:
- `key` - the time series key
- `timestamp` - Unix timestamp in milliseconds or `*` for current time
- `value` - a 64-bit floating point number

Returns an array of timestamps (one per sample), in the same order as the input.

## Basic Usage

```bash
# Add to multiple keys in one call
TS.MADD \
  cpu:host1 * 75.3 \
  memory:host1 * 4096 \
  disk:host1 * 87.2

# Returns:
# 1) (integer) 1700000000000
# 2) (integer) 1700000000000
# 3) (integer) 1700000000000
```

## Adding with Explicit Timestamps

```bash
TS.MADD \
  sensor:temp 1700000000000 22.5 \
  sensor:humidity 1700000000000 65.0 \
  sensor:pressure 1700000000000 1013.25

# Returns timestamps: 1700000000000, 1700000000000, 1700000000000
```

## Python Example - Single Host Metrics

```python
import redis
import psutil
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

# Initialize time series keys
hostname = 'host1'
ts.create(f'cpu:{hostname}', retention_msecs=86400000)
ts.create(f'memory:{hostname}', retention_msecs=86400000)
ts.create(f'disk:{hostname}', retention_msecs=86400000)

# Collect and push multiple metrics in one call
def collect_and_push(hostname):
    cpu = psutil.cpu_percent()
    memory = psutil.virtual_memory().percent
    disk = psutil.disk_usage('/').percent

    # Push all three metrics atomically
    timestamps = ts.madd([
        (f'cpu:{hostname}', '*', cpu),
        (f'memory:{hostname}', '*', memory),
        (f'disk:{hostname}', '*', disk),
    ])

    return timestamps

ts_result = collect_and_push('host1')
print(f"Inserted samples at: {ts_result}")
```

## Python Example - Multiple Hosts

```python
import redis
import random
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

hosts = ['host1', 'host2', 'host3']
metrics = ['cpu', 'memory']

def push_all_metrics(hosts, metrics):
    """Push metrics for all hosts in a single MADD call."""
    samples = []
    now = int(time.time() * 1000)

    for host in hosts:
        for metric in metrics:
            value = random.uniform(10, 90)
            samples.append((f'{metric}:{host}', now, value))

    timestamps = ts.madd(samples)
    return len(timestamps)

count = push_all_metrics(hosts, metrics)
print(f"Inserted {count} samples in one command")
```

## Handling Errors in Batch

`TS.MADD` returns an error for individual failed samples, but continues processing the rest:

```bash
# If key does not exist and is not auto-created
TS.MADD existing_series * 1.0 nonexistent_series * 2.0
# Returns:
# 1) (integer) 1700000000000   # OK
# 2) (error) TSDB: the key does not exist
```

In Python, check the result array for error entries:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def safe_madd(ts, samples):
    """Add multiple samples and report any errors."""
    try:
        results = ts.madd(samples)
        for i, (key, ts_val, value) in enumerate(samples):
            if isinstance(results[i], Exception):
                print(f"Error for {key}: {results[i]}")
            else:
                pass  # Success
        return results
    except Exception as e:
        print(f"MADD failed: {e}")
        return []
```

## Comparing TS.ADD vs TS.MADD

| Scenario | Use TS.ADD | Use TS.MADD |
|---|---|---|
| Single metric | Yes | Possible but overkill |
| Multiple metrics, same time | No | Yes |
| Multiple hosts | No | Yes |
| Pipeline optimization | Yes (with PIPELINE) | Yes (single RTT) |

## Backfilling Historical Data

```python
import redis
import random
from datetime import datetime, timedelta

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
ts = r.ts()

# Backfill 1 hour of data (1-second intervals) for 3 metrics
start = datetime(2024, 1, 15, 12, 0, 0)
samples = []

for i in range(3600):
    ts_ms = int((start + timedelta(seconds=i)).timestamp() * 1000)
    samples.append(('cpu:host1', ts_ms, random.uniform(20, 80)))
    samples.append(('memory:host1', ts_ms, random.uniform(30, 70)))

    # Send in batches of 200 triplets (100 per key)
    if len(samples) >= 200:
        ts.madd(samples)
        samples = []

if samples:
    ts.madd(samples)

print("Backfill complete")
```

## Summary

`TS.MADD` is the efficient way to ingest multiple time series samples in a single Redis round trip. It reduces network overhead and is ideal for monitoring agents that collect multiple metrics from one or more hosts simultaneously. Use it whenever you need to write correlated metrics at the same timestamp. Unlike pipeline batching, `TS.MADD` is a single atomic command at the Redis level, ensuring all samples in the batch are processed together.
