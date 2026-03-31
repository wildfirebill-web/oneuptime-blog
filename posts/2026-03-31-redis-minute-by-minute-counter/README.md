# How to Implement a Minute-by-Minute Counter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Time Series, Analytics, Metrics

Description: Build precise minute-by-minute counters in Redis using time-bucketed keys to track requests, events, or errors over time.

---

Minute-resolution counters are the foundation of many monitoring dashboards - requests per minute, errors per minute, sign-ups per minute. Redis makes these trivial to implement with atomic increments and automatic key expiry.

## The Time-Bucket Key Pattern

The key insight is encoding the timestamp bucket directly in the key name. All events in the same minute share the same key:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def minute_bucket(ts: float = None) -> str:
    ts = ts or time.time()
    # Truncate to minute boundary: YYYYMMDDHHMM
    return time.strftime("%Y%m%d%H%M", time.gmtime(ts))

def increment_counter(metric: str, value: int = 1) -> int:
    key = f"counter:{metric}:{minute_bucket()}"
    count = r.incrby(key, value)
    r.expire(key, 7200)  # keep 2 hours of minute data
    return count
```

## Reading the Last N Minutes

Reconstruct the time series by generating keys for past minutes:

```python
def get_last_n_minutes(metric: str, n: int = 60) -> list:
    now = time.time()
    results = []
    for i in range(n):
        ts = now - (i * 60)
        bucket = minute_bucket(ts)
        key = f"counter:{metric}:{bucket}"
        count = r.get(key) or "0"
        results.append({
            "minute": time.strftime("%H:%M", time.gmtime(ts)),
            "count": int(count),
        })
    return list(reversed(results))
```

## Batch Reading with Pipeline

Use a pipeline to read multiple minute buckets in one round trip:

```python
def get_minute_series(metric: str, n: int = 60) -> dict:
    now = time.time()
    pipe = r.pipeline()
    buckets = []
    for i in range(n):
        ts = now - (i * 60)
        bucket = minute_bucket(ts)
        key = f"counter:{metric}:{bucket}"
        pipe.get(key)
        buckets.append(time.strftime("%H:%M", time.gmtime(ts)))

    values = pipe.execute()
    return {
        b: int(v or 0)
        for b, v in zip(reversed(buckets), reversed(values))
    }
```

## Tracking Multiple Metrics Per Minute

Use a hash to track multiple dimensions in a single key:

```python
def increment_multi(minute: str = None):
    minute = minute or minute_bucket()
    key = f"stats:{minute}"
    pipe = r.pipeline()
    pipe.hincrby(key, "requests", 1)
    pipe.hincrby(key, "errors", 0)
    pipe.expire(key, 7200)
    pipe.execute()

def record_request(status_code: int):
    minute = minute_bucket()
    key = f"stats:{minute}"
    pipe = r.pipeline()
    pipe.hincrby(key, "requests", 1)
    if status_code >= 500:
        pipe.hincrby(key, "errors", 1)
    pipe.expire(key, 7200)
    pipe.execute()
```

## Computing Rates

Calculate requests per second from a minute counter:

```python
def get_current_rps(metric: str) -> float:
    key = f"counter:{metric}:{minute_bucket()}"
    count = int(r.get(key) or 0)
    elapsed = time.time() % 60  # seconds into current minute
    if elapsed == 0:
        return 0.0
    return count / elapsed
```

## Summary

Redis minute-by-minute counters use time-bucketed key names with atomic INCRBY operations and TTL-based expiry. Pipeline reads reconstruct time-series data efficiently in a single round trip. This pattern scales to millions of events per minute and serves as the raw data layer for real-time dashboards and alerting.
