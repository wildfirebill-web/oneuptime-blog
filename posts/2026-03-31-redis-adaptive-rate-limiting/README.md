# How to Implement Adaptive Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Performance

Description: Build an adaptive Redis rate limiter that automatically tightens limits during high load and relaxes them when the system recovers.

---

Static rate limits are either too restrictive during normal operation or insufficient during traffic spikes. Adaptive rate limiting monitors system load in real time and dynamically adjusts limits - tightening during high load to protect backend services and relaxing when load drops to maximize throughput.

## How Adaptive Rate Limiting Works

1. Track system health metrics (CPU, queue depth, response times) in Redis
2. Compute a current load factor (0.0 = healthy, 1.0 = overloaded)
3. Scale the rate limit inversely with the load factor
4. Re-evaluate the load factor on a configurable interval

## Load Metric Collection

Track request latency and error rate as proxy signals for backend health:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_request_latency(latency_ms: float):
    key = f"metrics:latency:{int(time.time() // 10)}"  # 10s buckets
    r.lpush(key, latency_ms)
    r.ltrim(key, 0, 999)  # Keep last 1000 samples
    r.expire(key, 60)

def get_avg_latency_ms() -> float:
    key = f"metrics:latency:{int(time.time() // 10)}"
    samples = r.lrange(key, 0, -1)
    if not samples:
        return 0.0
    return sum(float(s) for s in samples) / len(samples)

def record_error():
    key = f"metrics:errors:{int(time.time() // 60)}"
    r.incr(key)
    r.expire(key, 120)

def get_error_rate() -> int:
    key = f"metrics:errors:{int(time.time() // 60)}"
    return int(r.get(key) or 0)
```

## Computing the Load Factor

```python
LATENCY_THRESHOLD_MS = 200   # Normal expected latency
LATENCY_MAX_MS = 1000        # Overloaded latency
ERROR_RATE_THRESHOLD = 50    # Normal error count per minute

def compute_load_factor() -> float:
    avg_latency = get_avg_latency_ms()
    error_rate = get_error_rate()

    latency_factor = min(1.0, max(0.0,
        (avg_latency - LATENCY_THRESHOLD_MS) / (LATENCY_MAX_MS - LATENCY_THRESHOLD_MS)
    ))
    error_factor = min(1.0, error_rate / (ERROR_RATE_THRESHOLD * 5))

    return max(latency_factor, error_factor)
```

## Adaptive Rate Check

```python
BASE_LIMIT = 100
MIN_LIMIT = 10  # Never go below this

def get_adaptive_limit() -> int:
    load = compute_load_factor()
    # Scale limit down as load increases
    adaptive_limit = int(BASE_LIMIT * (1.0 - load * 0.9))
    return max(MIN_LIMIT, adaptive_limit)

def check_adaptive_rate_limit(identifier: str) -> dict:
    limit = get_adaptive_limit()
    load = compute_load_factor()
    window = int(time.time() // 60)
    key = f"ratelimit:adaptive:{identifier}:{window}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, 65)
    results = pipe.execute()

    count = results[0]
    return {
        "allowed": count <= limit,
        "limit": limit,
        "remaining": max(0, limit - count),
        "load_factor": round(load, 3),
    }
```

## Exposing Load State for Debugging

```python
def get_system_health() -> dict:
    return {
        "avg_latency_ms": round(get_avg_latency_ms(), 2),
        "error_rate_per_min": get_error_rate(),
        "load_factor": round(compute_load_factor(), 3),
        "current_rate_limit": get_adaptive_limit(),
    }
```

## Testing Adaptive Behavior

```bash
# Simulate high latency by injecting fake metrics
redis-cli LPUSH "metrics:latency:$(date +%s | awk '{print int($1/10)}')" 800 900 950

# Check how the limit adapts
python3 -c "from adaptive_limiter import get_adaptive_limit, compute_load_factor; print(f'load={compute_load_factor():.2f} limit={get_adaptive_limit()}')"
```

## Summary

Adaptive rate limiting protects your backend during traffic spikes by using real-time metrics stored in Redis to compute a dynamic limit that scales inversely with system load. By tracking latency and error rates in rolling time windows, the system automatically tightens limits as the backend degrades and relaxes them as it recovers - no manual tuning needed during incidents.
