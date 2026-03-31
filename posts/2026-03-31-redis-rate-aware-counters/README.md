# How to Implement Rate-Aware Counters with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Rate, Metrics, Analytics

Description: Build rate-aware counters in Redis that calculate events per second, per minute, and velocity trends from raw increment data.

---

A raw counter tells you how many events happened. A rate-aware counter tells you how fast they are happening right now. Request rate, error rate, and event velocity are all derived metrics that require rate calculation on top of raw counts.

## Instantaneous Rate from a Counter

Calculate current requests-per-second from a sliding time window:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_current_rate(metric: str, window_seconds: int = 60) -> float:
    now = time.time()
    window_start = now - window_seconds

    # Use a sorted set of timestamps as a sliding window
    key = f"rate_window:{metric}"
    r.zremrangebyscore(key, 0, window_start * 1000)
    count = r.zcard(key)

    return count / window_seconds  # events per second

def record_event_for_rate(metric: str):
    key = f"rate_window:{metric}"
    now_ms = int(time.time() * 1000)
    r.zadd(key, {str(now_ms): now_ms})
    r.expire(key, 120)
```

## Rate from Minute Buckets

A less memory-intensive approach computes rate from adjacent minute counters:

```python
def get_rate_per_minute(metric: str) -> dict:
    now = time.time()
    keys = []
    for i in range(5):
        ts = now - i * 60
        bucket = time.strftime("%Y%m%d%H%M", time.gmtime(ts))
        keys.append(f"counter:{metric}:{bucket}")

    pipe = r.pipeline()
    for key in keys:
        pipe.get(key)
    values = [int(v or 0) for v in pipe.execute()]

    current_minute = values[0]
    previous_minutes = values[1:5]
    avg_previous = sum(previous_minutes) / max(len(previous_minutes), 1)

    return {
        "current_rpm": current_minute,
        "avg_rpm_5min": avg_previous,
        "trend": "up" if current_minute > avg_previous * 1.1 else
                 "down" if current_minute < avg_previous * 0.9 else "stable",
    }
```

## Velocity Alert

Detect sudden spikes by comparing current rate to the rolling average:

```python
def check_velocity_alert(metric: str, threshold_multiplier: float = 3.0) -> bool:
    data = get_rate_per_minute(metric)
    current = data["current_rpm"]
    average = data["avg_rpm_5min"]

    if average == 0:
        return False

    return (current / average) > threshold_multiplier

def monitor_metric(metric: str):
    data = get_rate_per_minute(metric)
    print(f"{metric}: {data['current_rpm']} rpm (5min avg: {data['avg_rpm_5min']:.1f}) [{data['trend']}]")
    if check_velocity_alert(metric):
        print(f"  ALERT: {metric} rate spike detected!")
```

## Rates with Multiple Resolutions

Track rates at multiple timescales simultaneously:

```python
def record_with_rates(metric: str, value: int = 1):
    now = time.time()
    pipe = r.pipeline()

    # 1-second bucket
    s = int(now)
    pipe.incrby(f"rate:{metric}:1s:{s}", value)
    pipe.expire(f"rate:{metric}:1s:{s}", 120)

    # 1-minute bucket
    m = int(now // 60) * 60
    pipe.incrby(f"rate:{metric}:1m:{m}", value)
    pipe.expire(f"rate:{metric}:1m:{m}", 3600)

    pipe.execute()

def get_rps(metric: str) -> float:
    s = int(time.time()) - 1  # previous complete second
    v = r.get(f"rate:{metric}:1s:{s}")
    return float(v or 0)

def get_rpm(metric: str) -> float:
    m = int((time.time() - 60) // 60) * 60  # previous complete minute
    v = r.get(f"rate:{metric}:1m:{m}")
    return float(v or 0)
```

## Summary

Rate-aware counters in Redis derive velocity metrics from raw event counts using sliding windows, bucket comparisons, and multi-resolution recording. Velocity trend detection compares the current window to a rolling average, enabling real-time anomaly alerts. Multi-resolution tracking at seconds and minutes provides both instantaneous and smoothed rate views for dashboards.
