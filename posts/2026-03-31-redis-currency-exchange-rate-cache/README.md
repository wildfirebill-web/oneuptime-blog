# How to Implement Currency Exchange Rate Cache with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Finance, Currency, Cache, TTL

Description: Cache currency exchange rates in Redis with short TTLs, stale-while-revalidate logic, and circuit breakers to serve fast conversions even when the rate provider is unavailable.

---

Currency conversion is a hot path in fintech applications. Rate provider APIs introduce latency and rate limits that make direct calls on every transaction impractical. Redis acts as a high-performance cache layer with millisecond reads and automatic expiry when rates go stale.

## Storing Exchange Rates

Cache rates as a hash per base currency:

```python
import redis
import requests
import time

r = redis.Redis()

def refresh_rates(base_currency):
    resp = requests.get(f"https://api.exchangerates.io/latest?base={base_currency}", timeout=5)
    rates = resp.json()["rates"]
    key = f"rates:{base_currency}"
    pipe = r.pipeline()
    pipe.hset(key, mapping=rates)
    pipe.expire(key, 300)  # 5-minute TTL
    pipe.execute()
    return rates
```

Retrieve a single rate:

```python
def get_rate(base, target):
    rate = r.hget(f"rates:{base}", target)
    return float(rate) if rate else None
```

## Stale-While-Revalidate Pattern

Serve the cached rate immediately and refresh in the background if it is near expiry:

```python
import threading

def get_rate_with_revalidation(base, target):
    key = f"rates:{base}"
    rate = r.hget(key, target)
    ttl = r.ttl(key)

    if ttl < 60:  # Less than 60s left - start background refresh
        threading.Thread(target=refresh_rates, args=(base,), daemon=True).start()

    if rate:
        return float(rate)

    # Cache miss - fetch synchronously
    rates = refresh_rates(base)
    return float(rates.get(target, 0))
```

## Circuit Breaker

If the rate provider fails repeatedly, stop hammering it:

```python
MAX_FAILURES = 5
CIRCUIT_OPEN_TTL = 60

def is_circuit_open(base):
    return r.exists(f"rates:circuit_open:{base}") == 1

def record_provider_failure(base):
    key = f"rates:failures:{base}"
    count = r.incr(key)
    r.expire(key, 120)
    if count >= MAX_FAILURES:
        r.set(f"rates:circuit_open:{base}", 1, ex=CIRCUIT_OPEN_TTL)
```

Serve stale rates when the circuit is open:

```python
def safe_get_rate(base, target):
    if is_circuit_open(base):
        # Serve stale if available
        rate = r.hget(f"rates:{base}", target)
        return float(rate) if rate else None
    try:
        return get_rate_with_revalidation(base, target)
    except Exception:
        record_provider_failure(base)
        rate = r.hget(f"rates:{base}", target)
        return float(rate) if rate else None
```

## Conversion Function

Convert an amount using the cached rates:

```python
def convert(amount, from_currency, to_currency):
    if from_currency == to_currency:
        return amount
    rate = safe_get_rate(from_currency, to_currency)
    if rate is None:
        raise ValueError(f"No rate available for {from_currency}/{to_currency}")
    return round(amount * rate, 4)
```

## Historical Rate Snapshots

Preserve rates at fixed intervals for audit trails:

```python
def snapshot_rates(base_currency):
    rates = {k.decode(): v.decode() for k, v in r.hgetall(f"rates:{base_currency}").items()}
    hour = int(time.time() // 3600) * 3600
    r.hset(f"rates:snapshot:{base_currency}:{hour}", mapping=rates)
    r.expire(f"rates:snapshot:{base_currency}:{hour}", 86400 * 30)
```

## Summary

Redis delivers sub-millisecond exchange rate lookups through hash-based caching with TTL expiry, stale-while-revalidate background refreshes, and circuit breakers that protect against provider outages. Historical snapshots provide an audit trail for transaction reconstruction without adding overhead to the hot path.
