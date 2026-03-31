# How to Monitor Per-Tenant Redis Usage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Multi-Tenancy, Monitoring, Observability, SaaS

Description: Learn how to track per-tenant Redis key counts, memory usage, and command rates using keyspace scanning, counters, and custom metrics exported to your observability stack.

---

In a shared Redis deployment, understanding how much each tenant uses helps you enforce quotas, bill accurately, and detect runaway tenants before they affect others.

## Strategy 1: Key Count by Prefix Using SCAN

Use `SCAN` (never `KEYS`) to count keys per tenant prefix:

```python
import redis
from collections import Counter

r = redis.Redis(decode_responses=True)

def count_keys_by_tenant_prefix() -> dict:
    """Scan all keys and count by tenant prefix."""
    counts = Counter()
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, count=500)
        for key in keys:
            # Keys formatted as "tenant_id:type:identifier"
            prefix = key.split(":")[0]
            counts[prefix] += 1
        if cursor == 0:
            break
    return dict(counts)

# Run this on a low-traffic schedule (not in the hot path)
usage = count_keys_by_tenant_prefix()
for tenant, count in sorted(usage.items(), key=lambda x: -x[1]):
    print(f"{tenant}: {count} keys")
```

## Strategy 2: Track Usage Counters in Real Time

Maintain a per-tenant key count counter that updates on every write:

```python
def set_and_track(tenant_id: str, key: str, value: str):
    """Set a key and update the tenant's usage counter."""
    full_key = f"{tenant_id}:{key}"
    counter_key = f"stats:keys:{tenant_id}"

    existed = r.exists(full_key)
    r.set(full_key, value)
    if not existed:
        r.incr(counter_key)

def delete_and_track(tenant_id: str, key: str):
    full_key = f"{tenant_id}:{key}"
    counter_key = f"stats:keys:{tenant_id}"

    deleted = r.delete(full_key)
    if deleted:
        r.decr(counter_key)

def get_tenant_key_count(tenant_id: str) -> int:
    return int(r.get(f"stats:keys:{tenant_id}") or 0)
```

## Strategy 3: Estimate Per-Tenant Memory

Approximate memory per tenant by sampling key sizes:

```python
def estimate_tenant_memory(tenant_id: str, sample_size: int = 100) -> dict:
    """Sample keys for a tenant and extrapolate total memory usage."""
    keys = []
    cursor = 0
    while len(keys) < sample_size:
        cursor, batch = r.scan(cursor, match=f"{tenant_id}:*", count=100)
        keys.extend(batch)
        if cursor == 0:
            break

    if not keys:
        return {"tenant_id": tenant_id, "estimated_bytes": 0, "key_count": 0}

    total_bytes = sum(r.memory_usage(k) or 0 for k in keys[:sample_size])
    avg_bytes_per_key = total_bytes / len(keys[:sample_size])
    total_keys = get_tenant_key_count(tenant_id)
    estimated_total = avg_bytes_per_key * total_keys

    return {
        "tenant_id": tenant_id,
        "sampled_keys": len(keys[:sample_size]),
        "avg_bytes_per_key": round(avg_bytes_per_key, 1),
        "estimated_total_bytes": round(estimated_total),
        "estimated_total_mb": round(estimated_total / 1024 / 1024, 2)
    }
```

## Strategy 4: Track Command Rates Per Tenant

Increment a per-tenant command counter in your Redis middleware:

```python
import time

class MetricsRedis(redis.Redis):
    def execute_command(self, *args, **options):
        tenant_id = self._current_tenant_id  # Set via context var
        if tenant_id:
            window = int(time.time() // 60)  # Per-minute bucket
            r_metrics.incr(f"cmd_rate:{tenant_id}:{window}")
            r_metrics.expire(f"cmd_rate:{tenant_id}:{window}", 120)
        return super().execute_command(*args, **options)

def get_tenant_cmd_rate(tenant_id: str) -> int:
    """Get commands in the last minute."""
    window = int(time.time() // 60)
    prev_window = window - 1
    current = int(r.get(f"cmd_rate:{tenant_id}:{window}") or 0)
    previous = int(r.get(f"cmd_rate:{tenant_id}:{prev_window}") or 0)
    return current + previous
```

## Exporting Metrics to Prometheus

Expose per-tenant usage as Prometheus gauges:

```python
from prometheus_client import Gauge, start_http_server

KEY_COUNT = Gauge("redis_tenant_key_count", "Keys per tenant", ["tenant_id"])
CMD_RATE = Gauge("redis_tenant_cmd_rate_per_min", "Commands per minute per tenant", ["tenant_id"])

def update_metrics():
    for tenant_id in get_all_tenant_ids():
        KEY_COUNT.labels(tenant_id=tenant_id).set(get_tenant_key_count(tenant_id))
        CMD_RATE.labels(tenant_id=tenant_id).set(get_tenant_cmd_rate(tenant_id))

start_http_server(8000)
```

## Summary

Per-tenant Redis monitoring requires a combination of real-time counters updated on every write and periodic scans to verify accuracy. Exporting these metrics to Prometheus enables dashboards and alerting per tenant. Set threshold alerts in OneUptime when any tenant's key count or command rate exceeds their quota, allowing you to intervene before they degrade service for other tenants.
