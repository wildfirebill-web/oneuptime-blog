# How to Implement Anomaly Detection with RedisTimeSeries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, Anomaly Detection, Time Series

Description: Implement real-time anomaly detection on time-series metrics using RedisTimeSeries range queries and Z-score statistical analysis.

---

Anomaly detection on time-series data helps catch outages, traffic spikes, and unusual behavior before they impact users. RedisTimeSeries provides the fast range queries needed to compute rolling statistics for Z-score based anomaly detection in real time.

## Approach: Z-Score Detection

A Z-score measures how many standard deviations a value is from the mean:

```text
z = (x - mean) / std_dev
```

Values with |z| > 3 are typically flagged as anomalies.

## Setup

```python
import redis
import time
import math

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def setup_metric(metric_name: str, retention_ms: int = 86400000):
    try:
        r.execute_command(
            'TS.CREATE', f"metric:{metric_name}",
            'RETENTION', retention_ms,
            'LABELS', 'metric', metric_name
        )
    except Exception:
        pass

setup_metric("api_latency_ms")
```

## Recording Samples

```python
def record_sample(metric_name: str, value: float,
                  ts_ms: int = None):
    key = f"metric:{metric_name}"
    ts = ts_ms or int(time.time() * 1000)
    r.execute_command('TS.ADD', key, ts, value)
```

## Computing Rolling Statistics

Fetch a historical window and compute mean and standard deviation:

```python
def get_rolling_stats(metric_name: str,
                       window_minutes: int = 60) -> dict:
    key = f"metric:{metric_name}"
    now = int(time.time() * 1000)
    from_ts = now - window_minutes * 60 * 1000

    result = r.execute_command('TS.RANGE', key, from_ts, now)
    if len(result) < 10:
        return {}  # not enough data

    values = [float(val) for _, val in result]
    n = len(values)
    mean = sum(values) / n
    variance = sum((v - mean) ** 2 for v in values) / n
    std_dev = math.sqrt(variance)
    return {"mean": round(mean, 4), "std_dev": round(std_dev, 4), "n": n}

def compute_zscore(value: float, stats: dict) -> float:
    if not stats or stats.get('std_dev', 0) == 0:
        return 0.0
    return abs((value - stats['mean']) / stats['std_dev'])
```

## Real-Time Anomaly Check

```python
def check_anomaly(metric_name: str, value: float,
                   window_minutes: int = 60,
                   z_threshold: float = 3.0) -> dict:
    stats = get_rolling_stats(metric_name, window_minutes)
    if not stats:
        return {"is_anomaly": False, "reason": "insufficient_data"}

    z_score = compute_zscore(value, stats)
    is_anomaly = z_score > z_threshold

    return {
        "value": value,
        "mean": stats['mean'],
        "std_dev": stats['std_dev'],
        "z_score": round(z_score, 3),
        "is_anomaly": is_anomaly,
        "severity": "critical" if z_score > 5 else "warning" if is_anomaly else "normal"
    }

# Check latest reading
result = check_anomaly("api_latency_ms", 950.0)
if result["is_anomaly"]:
    print(f"ANOMALY: z={result['z_score']}, severity={result['severity']}")
```

## Automated Anomaly Scanner

Run as a background job to continuously check for anomalies:

```python
def scan_for_anomalies(metric_name: str,
                        window_minutes: int = 60) -> list:
    key = f"metric:{metric_name}"
    now = int(time.time() * 1000)
    # Check the last 5 minutes of data
    recent = r.execute_command(
        'TS.RANGE', key,
        now - 5 * 60 * 1000, now
    )

    stats = get_rolling_stats(metric_name, window_minutes)
    if not stats:
        return []

    anomalies = []
    for ts, val in recent:
        value = float(val)
        z = compute_zscore(value, stats)
        if z > 3.0:
            anomalies.append({
                "ts": int(ts),
                "value": value,
                "z_score": round(z, 3)
            })
    return anomalies
```

## Storing Anomaly Events

```python
def record_anomaly(metric_name: str, ts_ms: int,
                    value: float, z_score: float):
    r.lpush(f"anomalies:{metric_name}",
            f"{ts_ms}:{value}:{z_score}")
    r.ltrim(f"anomalies:{metric_name}", 0, 999)  # keep last 1000
```

## Summary

RedisTimeSeries enables real-time anomaly detection by providing fast range queries for computing rolling means and standard deviations. The Z-score approach is simple yet effective for detecting spikes and drops in metrics like latency, error rate, and throughput. Combine automated scanning with anomaly storage to build a lightweight alerting system that operates entirely within Redis.
