# How to Implement SLA Monitoring with RedisTimeSeries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisTimeSeries, SLA, Monitoring

Description: Track service SLA compliance by storing uptime and latency metrics in RedisTimeSeries and computing availability percentages over rolling windows.

---

SLA monitoring requires precise tracking of uptime, error budgets, and response time compliance. RedisTimeSeries stores these metrics efficiently and supports the time-range queries needed to compute SLA percentages over any window.

## SLA Metrics Design

For each service, track:
- **up**: 1 if healthy, 0 if down (checked every minute)
- **latency_ms**: measured response time per check
- **within_sla**: 1 if latency is within SLA threshold, 0 if not

## Setup

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

SLA_THRESHOLDS = {
    "api_gateway": {"latency_ms": 200, "availability": 99.9},
    "payment_service": {"latency_ms": 500, "availability": 99.95},
}

def create_sla_series(service: str):
    for metric in ["up", "latency_ms", "within_sla"]:
        key = f"sla:{service}:{metric}"
        try:
            r.execute_command(
                'TS.CREATE', key,
                'RETENTION', 90 * 86400000,  # 90 days
                'LABELS', 'service', service, 'metric', metric
            )
        except Exception:
            pass

for svc in SLA_THRESHOLDS:
    create_sla_series(svc)
```

## Recording Health Checks

```python
import requests as http_requests

def run_health_check(service: str, url: str,
                     timeout: float = 5.0) -> dict:
    ts = int(time.time() * 1000)
    threshold = SLA_THRESHOLDS.get(service, {}).get("latency_ms", 1000)

    try:
        start = time.monotonic()
        resp = http_requests.get(url, timeout=timeout)
        latency = (time.monotonic() - start) * 1000
        is_up = 1 if resp.status_code < 500 else 0
    except Exception:
        latency = timeout * 1000
        is_up = 0

    within_sla = 1 if (is_up == 1 and latency <= threshold) else 0

    pipe = r.pipeline()
    pipe.execute_command('TS.ADD', f"sla:{service}:up", ts, is_up)
    pipe.execute_command('TS.ADD', f"sla:{service}:latency_ms", ts, round(latency, 2))
    pipe.execute_command('TS.ADD', f"sla:{service}:within_sla", ts, within_sla)
    pipe.execute()

    return {"service": service, "up": is_up,
            "latency_ms": round(latency, 2), "within_sla": within_sla}
```

## Computing Availability

```python
def compute_availability(service: str, days: int = 30) -> float:
    now = int(time.time() * 1000)
    from_ts = now - days * 86400 * 1000

    # Sum all "up" readings and total count
    result = r.execute_command(
        'TS.RANGE', f"sla:{service}:up", from_ts, now
    )
    if not result:
        return 0.0

    total = len(result)
    up_count = sum(1 for _, val in result if float(val) == 1.0)
    return round(up_count / total * 100, 4)

def compute_latency_compliance(service: str, days: int = 30) -> float:
    now = int(time.time() * 1000)
    from_ts = now - days * 86400 * 1000

    result = r.execute_command(
        'TS.RANGE', f"sla:{service}:within_sla", from_ts, now
    )
    if not result:
        return 0.0

    total = len(result)
    compliant = sum(1 for _, val in result if float(val) == 1.0)
    return round(compliant / total * 100, 4)
```

## SLA Report

```python
def generate_sla_report(service: str, days: int = 30) -> dict:
    avail = compute_availability(service, days)
    latency_compliance = compute_latency_compliance(service, days)
    target = SLA_THRESHOLDS.get(service, {}).get("availability", 99.9)

    # Compute error budget consumed
    downtime_minutes = (100 - avail) / 100 * days * 24 * 60
    budget_total = (100 - target) / 100 * days * 24 * 60
    budget_consumed = round(downtime_minutes / budget_total * 100, 1) if budget_total > 0 else 0

    return {
        "service": service,
        "period_days": days,
        "availability_pct": avail,
        "sla_target_pct": target,
        "sla_met": avail >= target,
        "latency_compliance_pct": latency_compliance,
        "error_budget_consumed_pct": budget_consumed,
        "downtime_minutes": round(downtime_minutes, 1)
    }

print(generate_sla_report("api_gateway", days=30))
```

## Summary

RedisTimeSeries enables precise SLA monitoring by storing per-minute health check results with 90-day retention. Use TS.RANGE to compute availability percentages over any rolling window and compare against SLA targets. The error budget calculation helps teams understand how much downtime headroom remains before the SLA is breached.
