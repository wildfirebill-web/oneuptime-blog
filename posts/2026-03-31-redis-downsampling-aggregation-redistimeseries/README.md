# How to Implement Downsampling and Aggregation with RedisTimeSeries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Downsampling, Aggregation

Description: Implement automatic downsampling and time-range aggregation with RedisTimeSeries compaction rules to reduce storage while preserving analytical value.

---

High-frequency time-series data grows quickly and becomes expensive to store and query over long periods. RedisTimeSeries solves this with compaction rules: automatic background aggregation that continuously downsample raw data into coarser buckets.

## Core Concepts

- **Compaction rule**: a TS.CREATERULE that maps raw writes to an aggregated destination
- **Aggregation types**: avg, sum, min, max, count, first, last, range
- **Bucket alignment**: all samples within a time bucket are aggregated into one point

## Setup

```bash
docker run -p 6379:6379 redis/redis-stack-server:latest
pip install redis
```

## Setting Up a Multi-Resolution Pipeline

Create a raw series and multiple downsampled destinations:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_metric_pipeline(metric_name: str):
    raw_key = f"metric:{metric_name}:raw"
    resolutions = [
        # (suffix, bucket_ms, retention_ms, agg_type)
        ("1min",  60_000,          30 * 86400_000, "avg"),
        ("5min",  5 * 60_000,      90 * 86400_000, "avg"),
        ("1hour", 3600_000,       365 * 86400_000, "avg"),
        ("1day",  86400_000, 5 * 365 * 86400_000, "avg"),
    ]

    # Create raw series - 24 hours retention
    try:
        r.execute_command(
            'TS.CREATE', raw_key,
            'RETENTION', 86400_000,
            'LABELS', 'metric', metric_name, 'resolution', 'raw'
        )
    except Exception:
        pass

    for suffix, bucket_ms, retention_ms, agg in resolutions:
        dest_key = f"metric:{metric_name}:{suffix}"
        try:
            r.execute_command(
                'TS.CREATE', dest_key,
                'RETENTION', retention_ms,
                'LABELS', 'metric', metric_name, 'resolution', suffix
            )
            r.execute_command(
                'TS.CREATERULE', raw_key, dest_key,
                'AGGREGATION', agg, bucket_ms
            )
        except Exception:
            pass

create_metric_pipeline("api_response_time_ms")
create_metric_pipeline("request_count")
```

## Ingesting Raw Data

```python
import time

def record(metric_name: str, value: float, ts_ms: int = None):
    key = f"metric:{metric_name}:raw"
    ts = ts_ms or int(time.time() * 1000)
    r.execute_command('TS.ADD', key, ts, value)
```

## Querying with On-the-Fly Aggregation

For resolutions not covered by compaction rules, use TS.RANGE with AGGREGATION:

```python
def query_range(metric_name: str, from_ms: int, to_ms: int,
                bucket_ms: int, agg: str = "avg"):
    # First check if we have a matching compaction key
    resolution_map = {
        60_000: "1min",
        300_000: "5min",
        3600_000: "1hour",
        86400_000: "1day"
    }

    if bucket_ms in resolution_map:
        key = f"metric:{metric_name}:{resolution_map[bucket_ms]}"
    else:
        # Fall back to raw with on-the-fly aggregation
        key = f"metric:{metric_name}:raw"

    result = r.execute_command(
        'TS.RANGE', key, from_ms, to_ms,
        'AGGREGATION', agg, bucket_ms
    )
    return [{"ts": int(ts), "value": float(val)} for ts, val in result]
```

## Multiple Aggregation Functions on the Same Series

Create separate compaction destinations for different aggregation types:

```python
def create_multi_agg_pipeline(metric_name: str, bucket_ms: int):
    raw_key = f"metric:{metric_name}:raw"
    bucket_label = f"{bucket_ms // 1000}s"

    for agg in ["avg", "min", "max", "count"]:
        dest_key = f"metric:{metric_name}:{bucket_label}:{agg}"
        try:
            r.execute_command(
                'TS.CREATE', dest_key,
                'RETENTION', 30 * 86400_000,
                'LABELS', 'metric', metric_name,
                'resolution', bucket_label, 'agg', agg
            )
            r.execute_command(
                'TS.CREATERULE', raw_key, dest_key,
                'AGGREGATION', agg, bucket_ms
            )
        except Exception:
            pass

# Create avg, min, max, count for 1-minute buckets
create_multi_agg_pipeline("api_response_time_ms", 60_000)
```

## Viewing Compaction Rules

```bash
TS.INFO metric:api_response_time_ms:raw
```

The `rules` field in the output lists all compaction rules associated with the series.

## Summary

RedisTimeSeries compaction rules provide a powerful automatic downsampling system that continuously reduces raw data into multi-resolution aggregates as new samples arrive. Create a pipeline with raw, minute, hour, and day buckets to cover all dashboard and reporting time ranges. Use on-the-fly aggregation for ad-hoc queries or unusual bucket sizes not covered by existing compaction rules.
