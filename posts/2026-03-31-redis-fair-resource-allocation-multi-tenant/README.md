# How to Implement Fair Resource Allocation in Multi-Tenant Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Multi-Tenancy, Rate Limiting, Architecture, SaaS

Description: Learn how to implement fair resource allocation in a shared Redis instance using per-tenant rate limiting, key quotas, and memory budgets to prevent noisy-neighbor problems.

---

In a shared Redis deployment, one high-traffic tenant can consume all available CPU and memory, degrading service for all others. Fair resource allocation techniques prevent this noisy-neighbor problem.

## The Noisy Neighbor Problem

Without controls, one tenant can:

- Execute millions of commands per second, consuming all CPU
- Store gigabytes of data, exhausting available memory
- Open hundreds of connections, hitting the connection limit

## Technique 1: Per-Tenant Command Rate Limiting

Implement application-side rate limiting per tenant using Redis itself:

```python
import redis
import time

r = redis.Redis()

def check_tenant_rate_limit(tenant_id: str, max_commands_per_second: int = 1000) -> bool:
    """Returns True if within limit, False if over limit."""
    key = f"rate:{tenant_id}:{int(time.time())}"
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, 2)
    result = pipe.execute()
    current_count = result[0]
    return current_count <= max_commands_per_second

# In your application layer
def execute_tenant_command(tenant_id: str, command_fn):
    if not check_tenant_rate_limit(tenant_id):
        raise Exception(f"Rate limit exceeded for tenant {tenant_id}")
    return command_fn()
```

## Technique 2: Per-Tenant Key Count Quotas

Track and enforce key quotas per tenant:

```python
def set_with_quota(tenant_id: str, key: str, value: str, max_keys: int = 10000):
    quota_key = f"quota:keys:{tenant_id}"

    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(quota_key)
                current_count = int(pipe.get(quota_key) or 0)

                if current_count >= max_keys:
                    raise Exception(f"Key quota exceeded for tenant {tenant_id}")

                pipe.multi()
                pipe.set(key, value)
                pipe.incr(quota_key)
                pipe.execute()
                break
            except redis.WatchError:
                continue

def delete_and_decrement(tenant_id: str, key: str):
    quota_key = f"quota:keys:{tenant_id}"
    with r.pipeline() as pipe:
        pipe.delete(key)
        pipe.decr(quota_key)
        pipe.execute()
```

## Technique 3: Per-Tenant Memory Budgets

Approximate per-tenant memory by tracking value sizes:

```python
def set_with_memory_budget(
    tenant_id: str,
    key: str,
    value: str,
    max_memory_bytes: int = 50 * 1024 * 1024  # 50MB
):
    memory_key = f"budget:memory:{tenant_id}"
    value_size = len(value.encode())

    current_usage = int(r.get(memory_key) or 0)
    if current_usage + value_size > max_memory_bytes:
        raise Exception(f"Memory budget exceeded for tenant {tenant_id}")

    pipe = r.pipeline()
    pipe.set(key, value)
    pipe.incrby(memory_key, value_size)
    pipe.execute()
```

## Technique 4: Separate Connection Pools Per Tenant

Limit the number of simultaneous connections each tenant can use:

```python
from redis.connection import ConnectionPool

TENANT_POOLS = {}
MAX_CONNECTIONS_PER_TENANT = 20

def get_pool_for_tenant(tenant_id: str) -> ConnectionPool:
    if tenant_id not in TENANT_POOLS:
        TENANT_POOLS[tenant_id] = ConnectionPool(
            host="localhost",
            port=6379,
            max_connections=MAX_CONNECTIONS_PER_TENANT,
            decode_responses=True
        )
    return TENANT_POOLS[tenant_id]

def get_redis_for_tenant(tenant_id: str) -> redis.Redis:
    pool = get_pool_for_tenant(tenant_id)
    return redis.Redis(connection_pool=pool)
```

## Monitoring Per-Tenant Resource Usage

Regularly audit which tenants are consuming the most resources:

```bash
#!/bin/bash
# Report per-tenant key counts from prefixed keyspace
redis-cli --scan --pattern "*:*" | \
  awk -F: '{print $1}' | \
  sort | uniq -c | sort -rn | head -20
```

Set alerts in OneUptime when a tenant's rate limit bucket hits 90% of capacity, giving you early warning before they impact other tenants.

## Summary

Fair resource allocation in shared Redis requires application-level enforcement since Redis itself has no built-in per-user quotas. Combining per-tenant rate limiting, key quotas, and connection pool limits creates a layered approach to preventing noisy-neighbor problems. Monitor usage per tenant regularly and alert when any tenant approaches their limits to prevent cascading failures across your multi-tenant deployment.
