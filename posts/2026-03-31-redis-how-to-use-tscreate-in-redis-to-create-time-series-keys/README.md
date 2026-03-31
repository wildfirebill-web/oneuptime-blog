# How to Use TS.CREATE in Redis to Create Time Series Keys

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Time Series, Metric, Monitoring

Description: Learn how to use TS.CREATE in Redis to create time series keys with retention policies, labels, and compaction rules for storing metrics data.

---

## What Is TS.CREATE?

`TS.CREATE` is a RedisTimeSeries command that creates a new time series key in Redis. Time series keys are purpose-built for storing timestamped numerical data (metrics, sensor readings, financial data) with built-in support for automatic sample retention, downsampling via compaction rules, and metadata labels.

RedisTimeSeries is available as part of Redis Stack or as a separate module.

## Basic Syntax

```text
TS.CREATE key
  [RETENTION retentionPeriod]
  [ENCODING [COMPRESSED | UNCOMPRESSED]]
  [CHUNK_SIZE chunkSize]
  [DUPLICATE_POLICY policy]
  [LABELS {label value}...]
```

Key options:
- `RETENTION` - how long to keep samples in milliseconds (0 = forever)
- `ENCODING` - COMPRESSED (default) or UNCOMPRESSED
- `CHUNK_SIZE` - memory chunk size in bytes (default: 4096)
- `DUPLICATE_POLICY` - what to do with duplicate timestamps
- `LABELS` - key-value metadata attached to the time series

## Creating a Basic Time Series

```bash
# Minimal time series key
TS.CREATE temperature:sensor1
# Returns: OK

# Add samples to it
TS.ADD temperature:sensor1 * 22.5
TS.ADD temperature:sensor1 * 23.1
```

## Creating with Retention Policy

```bash
# Keep data for 7 days (7 * 24 * 60 * 60 * 1000 ms)
TS.CREATE temperature:sensor1 RETENTION 604800000

# Keep data for 30 days
TS.CREATE metrics:cpu:host1 RETENTION 2592000000

# Keep forever (default)
TS.CREATE metrics:total RETENTION 0
```

## Creating with Labels

Labels are metadata key-value pairs attached to the time series. They are used for filtering with `TS.MRANGE` and `TS.QUERYINDEX`.

```bash
# CPU metrics for a specific host and region
TS.CREATE metrics:cpu:host1 \
  RETENTION 604800000 \
  LABELS host host1 region us-east env production metric cpu

# Memory metrics
TS.CREATE metrics:memory:host1 \
  RETENTION 604800000 \
  LABELS host host1 region us-east env production metric memory
```

## Duplicate Timestamp Policies

When the same timestamp is added twice, the DUPLICATE_POLICY controls behavior:

```bash
# BLOCK - reject duplicate timestamps (default)
TS.CREATE sensor:temp DUPLICATE_POLICY BLOCK

# LAST - keep the last value at a timestamp
TS.CREATE sensor:temp DUPLICATE_POLICY LAST

# FIRST - keep the first value at a timestamp
TS.CREATE sensor:temp DUPLICATE_POLICY FIRST

# MAX - keep the maximum value at a timestamp
TS.CREATE sensor:temp DUPLICATE_POLICY MAX

# MIN - keep the minimum value at a timestamp
TS.CREATE sensor:temp DUPLICATE_POLICY MIN

# SUM - add values at duplicate timestamps
TS.CREATE sensor:temp DUPLICATE_POLICY SUM
```

## Choosing Encoding

```bash
# Compressed (default) - best for most use cases, reduces memory
TS.CREATE metrics:cpu ENCODING COMPRESSED

# Uncompressed - faster writes, higher memory usage
TS.CREATE metrics:fast ENCODING UNCOMPRESSED
```

## Python Example

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Create time series for CPU and memory per host
hosts = ['host1', 'host2', 'host3']
metrics = ['cpu', 'memory', 'disk']
retention_ms = 7 * 24 * 60 * 60 * 1000  # 7 days

for host in hosts:
    for metric in metrics:
        key = f'metrics:{metric}:{host}'
        r.ts().create(
            key,
            retention_msecs=retention_ms,
            labels={
                'host': host,
                'metric': metric,
                'env': 'production',
            },
            duplicate_policy='LAST'
        )
        print(f"Created: {key}")
```

## Checking Time Series Info After Creation

```bash
TS.INFO metrics:cpu:host1
```

```text
1) totalSamples
2) (integer) 0
3) memoryUsage
4) (integer) 4256
5) firstTimestamp
6) (integer) 0
7) lastTimestamp
8) (integer) 0
9) retentionTime
10) (integer) 604800000
11) chunkCount
12) (integer) 1
...
```

## Creating a Time Series with Compaction Rules

For long-term storage with downsampled aggregates:

```bash
# Raw 1-second data (keep 1 hour)
TS.CREATE sensor:temp RETENTION 3600000

# 1-minute aggregated (keep 1 day)
TS.CREATE sensor:temp:1min RETENTION 86400000

# Create compaction rule: AVG over 60-second windows
TS.CREATERULE sensor:temp sensor:temp:1min AGGREGATION AVG 60000
```

## Summary

`TS.CREATE` is the foundation of RedisTimeSeries usage, allowing you to define time series keys with precise retention policies, metadata labels, and data encoding preferences. Always attach labels when creating time series to enable powerful multi-series queries with `TS.MRANGE` and `TS.QUERYINDEX`. Set appropriate retention periods to automatically expire old data and prevent unbounded memory growth, and consider adding compaction rules for long-term trend analysis.
