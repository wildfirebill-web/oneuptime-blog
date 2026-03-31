# How to Implement a Moving Average Calculator in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Moving Average, Analytics

Description: Build an efficient moving average calculator with Redis lists and sorted sets to compute rolling metrics for time-series data in real time.

---

Moving averages smooth out noise in time-series data and are essential for dashboards, anomaly detection, and financial analytics. Redis lets you maintain rolling windows of data points and compute moving averages in real time without storing full history.

## Simple Moving Average with a Capped List

Use a Redis list as a fixed-size window. Push new values and trim the list to keep only the last N values:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def push_value(key: str, value: float, window_size: int):
    pipe = r.pipeline()
    pipe.lpush(key, value)
    pipe.ltrim(key, 0, window_size - 1)  # Keep only the last N values
    pipe.execute()

def get_moving_average(key: str) -> float | None:
    values = r.lrange(key, 0, -1)
    if not values:
        return None
    floats = [float(v) for v in values]
    return sum(floats) / len(floats)

# Track average response time (last 60 samples)
def record_response_time(service: str, latency_ms: float):
    push_value(f"latency:{service}", latency_ms, window_size=60)

def get_avg_latency(service: str) -> float | None:
    return get_moving_average(f"latency:{service}")
```

## Time-Based Moving Average

For a true time-based window (e.g., last 5 minutes), use a sorted set with timestamps as scores:

```python
import time

def record_metric(key: str, value: float, window_seconds: int = 300):
    now = time.time()
    pipe = r.pipeline()
    # Add value with timestamp as score
    pipe.zadd(key, {f"{now}:{value}": now})
    # Remove values outside the window
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    pipe.expire(key, window_seconds + 10)
    pipe.execute()

def get_time_window_average(key: str, window_seconds: int = 300) -> float | None:
    now = time.time()
    r.zremrangebyscore(key, 0, now - window_seconds)
    members = r.zrange(key, 0, -1)
    if not members:
        return None
    # Extract value from "timestamp:value" format
    values = [float(m.split(":", 1)[1]) for m in members]
    return sum(values) / len(values)
```

## Exponential Moving Average

EMA gives more weight to recent values and is memory-efficient (only stores the current EMA):

```python
def update_ema(key: str, new_value: float, alpha: float = 0.1) -> float:
    """
    alpha: smoothing factor (0 < alpha < 1)
    Higher alpha = more weight on recent values.
    """
    current_ema = r.get(key)
    if current_ema is None:
        # Initialize with first value
        r.set(key, new_value)
        return new_value

    ema = alpha * new_value + (1 - alpha) * float(current_ema)
    r.set(key, ema)
    return ema

# Track EMA of CPU usage
def record_cpu(host: str, cpu_pct: float) -> float:
    return update_ema(f"ema:cpu:{host}", cpu_pct, alpha=0.2)
```

## Multi-Window Moving Average

Maintain several window sizes simultaneously for dashboard display:

```python
WINDOWS = {"1m": 60, "5m": 300, "15m": 900}

def record_multi_window(metric: str, value: float):
    pipe = r.pipeline(transaction=False)
    for label, window in WINDOWS.items():
        key = f"ma:{metric}:{label}"
        pipe.lpush(key, value)
        pipe.ltrim(key, 0, window - 1)
    pipe.execute()

def get_all_averages(metric: str) -> dict:
    result = {}
    for label in WINDOWS:
        key = f"ma:{metric}:{label}"
        values = r.lrange(key, 0, -1)
        if values:
            result[label] = sum(float(v) for v in values) / len(values)
    return result
```

## Summary

Redis lists are the simplest way to maintain a fixed-size sample window for computing simple moving averages. Sorted sets enable true time-based windows, and a single EMA key offers memory-efficient exponential smoothing. Together these patterns cover most real-time analytics use cases without a dedicated time-series database.
