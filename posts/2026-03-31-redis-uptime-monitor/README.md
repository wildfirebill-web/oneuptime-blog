# How to Build an Uptime Monitor with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Uptime Monitor, Monitoring, Availability, Backend

Description: Build a Redis-backed uptime monitor that stores check results in time-bucketed sorted sets, calculates rolling uptime percentages, and triggers alerts on threshold breaches.

---

An uptime monitor periodically checks whether services respond correctly and tracks the history of those checks. Redis time-series data stored in sorted sets lets you calculate uptime percentages over any window with a single range query.

## Recording Check Results

A probe runs every minute and records the result as a 1 (up) or 0 (down) in a sorted set scored by Unix timestamp:

```python
import redis
import time
import requests

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
CHECK_HISTORY_TTL = 90 * 24 * 3600  # 90 days

def record_check(service_name: str, is_up: bool, response_ms: int = 0):
    ts = time.time()
    key = f"uptime:{service_name}:checks"
    payload = f"{1 if is_up else 0}:{response_ms}"
    r.zadd(key, {f"{ts}:{payload}": ts})
    r.expire(key, CHECK_HISTORY_TTL)

def probe_service(service_name: str, url: str):
    try:
        start = time.time()
        resp = requests.get(url, timeout=10)
        elapsed_ms = int((time.time() - start) * 1000)
        record_check(service_name, resp.status_code < 500, elapsed_ms)
    except Exception:
        record_check(service_name, False, 0)
```

## Calculating Uptime Percentage

Query the sorted set for a time window and compute the ratio of up checks:

```python
def calculate_uptime(service_name: str, hours: int = 24) -> float:
    now = time.time()
    window_start = now - hours * 3600
    key = f"uptime:{service_name}:checks"
    checks = r.zrangebyscore(key, window_start, now)
    if not checks:
        return None
    up_count = sum(1 for c in checks if c.split(":")[1] == "1")
    return round(up_count / len(checks) * 100, 2)
```

## Tracking Current Status

Cache the current status separately for fast lookups:

```python
def update_current_status(service_name: str, is_up: bool):
    status = "up" if is_up else "down"
    current = r.hget(f"uptime:{service_name}:current", "status")
    if current != status:
        # Status changed - record transition time
        r.hset(f"uptime:{service_name}:current", mapping={
            "status": status,
            "since": str(time.time()),
        })

def get_current_status(service_name: str) -> dict:
    data = r.hgetall(f"uptime:{service_name}:current") or {}
    uptime_24h = calculate_uptime(service_name, 24)
    return {
        "service": service_name,
        "status": data.get("status", "unknown"),
        "since": data.get("since"),
        "uptime_24h": uptime_24h,
    }
```

## Alert on Consecutive Failures

Use a counter to detect consecutive failures and trigger alerts:

```python
ALERT_THRESHOLD = 3

def check_and_alert(service_name: str, is_up: bool):
    fail_key = f"uptime:{service_name}:consecutive_failures"
    if is_up:
        r.delete(fail_key)
    else:
        failures = r.incr(fail_key)
        r.expire(fail_key, 3600)
        if failures == ALERT_THRESHOLD:
            send_alert(service_name, f"Service down for {failures} consecutive checks")
```

## Summary

Redis sorted sets with Unix timestamp scores create an efficient time-series store for uptime check results. Range queries by time window produce accurate uptime percentages in O(log N + M) time. Combining time-series history with a cached current status and a consecutive failure counter gives you a complete uptime monitoring solution.

