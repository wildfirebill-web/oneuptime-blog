# How to Implement Global vs Local Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Distributed System

Description: Understand the tradeoffs between global and local Redis rate limiting and implement both patterns for distributed multi-instance deployments.

---

When your API runs on multiple instances, rate limiting gets more complex. Local rate limiting (per-instance counters in memory) is fast but allows users to exceed limits across instances. Global rate limiting (shared Redis counters) enforces accurate limits but adds network latency. Understanding when to use each is essential.

## Local Rate Limiting

Each instance maintains its own counter. A user with a 100 RPM limit could make 100 requests to each of 10 instances - effectively 1000 RPM:

```python
import time
from collections import defaultdict

# In-memory counter per instance
local_counters = defaultdict(lambda: {"count": 0, "window": 0})

def local_rate_limit(identifier: str, limit: int = 100, window: int = 60) -> bool:
    now_window = int(time.time() // window)
    state = local_counters[identifier]

    if state["window"] != now_window:
        state["count"] = 0
        state["window"] = now_window

    state["count"] += 1
    return state["count"] <= limit
```

Use case: intra-instance protection (e.g., throttling outbound calls from a single worker)

## Global Rate Limiting with Redis

All instances share a single Redis counter. Accurate but adds a round-trip:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def global_rate_limit(identifier: str, limit: int = 100, window: int = 60) -> bool:
    window_key = int(time.time() // window)
    key = f"global_rl:{identifier}:{window_key}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window + 5)
    results = pipe.execute()

    return results[0] <= limit
```

Use case: enforcing per-user or per-API-key limits across all instances

## Hybrid: Local Pre-Check + Global Enforcement

Use local counters for fast rejection of obvious excess traffic, then validate with Redis for the accurate count:

```python
# Local limits are intentionally more permissive (1/N of global limit)
NUM_INSTANCES = 5
LOCAL_LIMIT_FRACTION = 0.5  # Allow local burst up to 50% of per-instance share

def hybrid_rate_limit(
    identifier: str,
    global_limit: int = 100,
    window: int = 60,
    num_instances: int = NUM_INSTANCES
) -> bool:
    local_limit = int((global_limit / num_instances) * (1 + LOCAL_LIMIT_FRACTION))

    # Fast local check first
    if not local_rate_limit(identifier, limit=local_limit, window=window):
        return False  # Definitely over limit, skip Redis call

    # Accurate global check
    return global_rate_limit(identifier, limit=global_limit, window=window)
```

## When to Use Each Approach

```text
Pattern      | Accuracy | Latency | Use When
-------------|----------|---------|------------------------------------------
Local only   | Low      | None    | Internal throttling, single-instance apps
Global only  | Exact    | ~1ms    | Paid API quotas, security-critical limits
Hybrid       | Near-exact | <1ms  | High-traffic APIs, cost/accuracy balance
```

## Configuring Redis Connection Pool for Low Latency

```python
connection_pool = redis.ConnectionPool(
    host='redis-primary',
    port=6379,
    max_connections=20,
    socket_connect_timeout=0.05,  # 50ms connect timeout
    socket_timeout=0.1,           # 100ms operation timeout
)
r = redis.Redis(connection_pool=connection_pool)
```

## Monitoring Global vs Local Divergence

```bash
# Global counter for a user
redis-cli GET "global_rl:user_123:$(date +%s | awk '{print int($1/60)}')"

# If value is much lower than you'd expect, local counters may be rejecting before Redis
```

## Summary

Choose global Redis rate limiting when accuracy matters - paid quotas, security limits, API contracts. Choose local limiting when you need zero-latency rejection and can tolerate some over-allowance. The hybrid pattern delivers the best tradeoff for high-traffic production APIs: fast rejection for obvious excess traffic with Redis as the authoritative validator for accurate enforcement.
