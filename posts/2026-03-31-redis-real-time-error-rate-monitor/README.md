# How to Build a Real-Time Error Rate Monitor with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Error Rate, Monitoring, Real-Time, Alerting

Description: Build a real-time error rate monitor with Redis that tracks errors and successes per service and triggers alerts when thresholds are breached.

---

Monitoring error rates in real-time helps you catch incidents before they escalate. Redis provides the atomic counters and fast reads needed to compute error rates continuously without querying a slow database.

## Recording Requests and Errors

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_request(service: str, success: bool):
    now = int(time.time())
    minute = now // 60
    key = f"errors:{service}:min:{minute}"

    pipe = r.pipeline()
    pipe.hincrby(key, "total", 1)
    if not success:
        pipe.hincrby(key, "errors", 1)
    pipe.expire(key, 3600)
    pipe.execute()
```

## Computing Real-Time Error Rate

```python
def get_error_rate(service: str, window_minutes: int = 5) -> dict:
    now = int(time.time())
    current_minute = now // 60

    total = 0
    errors = 0

    pipe = r.pipeline()
    buckets = [current_minute - i for i in range(window_minutes)]
    for b in buckets:
        pipe.hmget(f"errors:{service}:min:{b}", "total", "errors")
    results = pipe.execute()

    for result in results:
        total += int(result[0] or 0)
        errors += int(result[1] or 0)

    rate = (errors / total * 100) if total > 0 else 0.0
    return {"total": total, "errors": errors, "error_rate_pct": round(rate, 2)}
```

## Threshold-Based Alerting

```python
ALERT_THRESHOLDS = {
    "api-gateway": 1.0,    # 1% error rate
    "checkout":    0.5,    # 0.5%
    "auth":        2.0,    # 2%
}

def check_and_alert(service: str) -> bool:
    threshold = ALERT_THRESHOLDS.get(service, 1.0)
    stats = get_error_rate(service)

    if stats["error_rate_pct"] > threshold:
        alert_key = f"alert:{service}:fired"
        # Only alert once per 5-minute window
        if not r.exists(alert_key):
            r.setex(alert_key, 300, "1")
            return True  # Trigger alert
    return False
```

## Service Error Rate Dashboard

```python
def get_all_service_error_rates(services: list) -> list:
    return [
        {"service": s, **get_error_rate(s)}
        for s in services
    ]
```

## Tracking Error Types

```python
def record_error_type(service: str, error_code: str):
    today = time.strftime("%Y-%m-%d")
    r.zincrby(f"errors:{service}:types:{today}", 1, error_code)
    r.expire(f"errors:{service}:types:{today}", 7 * 86400)

def get_top_errors(service: str, n: int = 10) -> list:
    today = time.strftime("%Y-%m-%d")
    return r.zrevrange(f"errors:{service}:types:{today}", 0, n - 1, withscores=True)
```

## Integration with OneUptime

[OneUptime](https://oneuptime.com) can consume your Redis-derived error rates via API and create incidents automatically when thresholds are breached.

```bash
# Verify error counter health
redis-cli HGETALL "errors:api-gateway:min:$(date +%s | awk '{print int($1/60)}')"
```

## Summary

Per-minute Hash buckets with HINCRBY allow atomic multi-field updates (total and errors) in a single pipeline call. Rolling window aggregation across multiple minute keys gives you a continuously updated error rate. Combining Redis-side deduplication (setex alert key) with your alerting system prevents alert storms.
