# How to Design a URL Shortener Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, URL Shortener, Interview, Architecture

Description: Walk through designing a URL shortener with Redis covering key generation, redirect caching, analytics counters, and scaling to billions of URLs.

---

## Requirements Clarification

Start every system design interview by clarifying requirements.

**Functional:**
- Create a short URL from a long URL
- Redirect short URL to original URL
- Optional: custom aliases, expiration, analytics

**Non-functional:**
- 100M URLs created per day
- 10B redirects per day (100:1 read-to-write ratio)
- P99 redirect latency under 10ms
- 99.99% availability

## Scale Estimation

```text
Write QPS: 100M / 86400 = ~1,160 writes/sec
Read QPS:  10B  / 86400 = ~115,700 reads/sec
Storage per URL: ~500 bytes
Total URLs (5 years): 100M * 365 * 5 = 182.5B URLs
Total storage: 182.5B * 500 = ~91TB
```

## High-Level Architecture

```text
[Client]
   |
[Load Balancer]
   |
[API Servers] --> [Redis Cache] --> [Primary DB (Postgres/Cassandra)]
   |
[ID Generator Service]
```

## Short URL Key Generation

A 7-character base62 string gives 62^7 = 3.5 trillion combinations - more than enough.

**Option 1: Counter-based with Redis INCR**

```bash
redis-cli INCR url:global_counter
```

Convert the counter to base62:

```python
import string

BASE62 = string.digits + string.ascii_letters  # 0-9a-zA-Z

def encode_base62(num: int) -> str:
    if num == 0:
        return BASE62[0]
    result = []
    while num:
        result.append(BASE62[num % 62])
        num //= 62
    return ''.join(reversed(result)).zfill(7)

def generate_short_code(r) -> str:
    counter = r.incr('url:global_counter')
    return encode_base62(counter)
```

Redis `INCR` is atomic and handles high concurrency safely. Allocate counter ranges to each API server to reduce Redis calls:

```python
# Allocate 1000 IDs to this server at once
start = r.incrby('url:global_counter', 1000)
local_range = range(start - 999, start + 1)
```

**Option 2: Hash-based**

Hash the long URL with MD5/SHA256 and take the first 7 characters of the base62 encoding. Simpler but requires collision handling.

## Storing URL Mappings in Redis

Store short-to-long URL mappings as Redis strings with optional TTL:

```bash
# Store mapping (1 year TTL)
redis-cli SET url:abc1234 "https://www.example.com/very/long/path" EX 31536000

# Retrieve for redirect
redis-cli GET url:abc1234
```

For custom aliases, also store reverse mapping:

```bash
redis-cli SET url_reverse:"https://www.example.com/very/long/path" "abc1234"
```

## Caching Strategy

With 115,700 reads/sec, the primary DB cannot handle all redirects directly. Use Redis as a read-through cache:

```python
import redis

r = redis.Redis(host='redis-cluster', port=6379)

def get_long_url(short_code: str) -> str | None:
    cache_key = f"url:{short_code}"

    # Check Redis cache first
    long_url = r.get(cache_key)
    if long_url:
        return long_url.decode()

    # Cache miss - query database
    long_url = db.query("SELECT long_url FROM urls WHERE short_code = %s", short_code)
    if long_url:
        # Cache with 24-hour TTL
        r.setex(cache_key, 86400, long_url)

    return long_url
```

80% of traffic typically hits 20% of URLs (Pareto principle). A Redis cache with 10GB RAM can hold ~20M URLs and serve the vast majority of traffic from cache.

## Analytics: Click Counting

Track clicks per short URL using Redis counters:

```bash
redis-cli INCR url_clicks:abc1234
```

For time-series analytics, use a sorted set or Redis Streams:

```python
import time

def record_click(r, short_code: str, metadata: dict):
    # Increment total counter
    r.incr(f"url_clicks:{short_code}")

    # Track clicks per hour using sorted set
    hour_bucket = int(time.time() / 3600)
    r.zincrby(f"url_hourly:{short_code}", 1, str(hour_bucket))
    r.expire(f"url_hourly:{short_code}", 7 * 86400)  # Keep 7 days

def get_click_count(r, short_code: str) -> int:
    return int(r.get(f"url_clicks:{short_code}") or 0)

def get_hourly_clicks(r, short_code: str, hours: int = 24) -> list:
    now_bucket = int(time.time() / 3600)
    min_bucket = now_bucket - hours
    results = r.zrangebyscore(
        f"url_hourly:{short_code}",
        min_bucket,
        now_bucket,
        withscores=True
    )
    return results
```

## Rate Limiting URL Creation

Prevent abuse by rate limiting URL creation per user:

```python
def can_create_url(r, user_id: str, limit: int = 100) -> bool:
    window = int(time.time() / 60)
    key = f"ratelimit:create:{user_id}:{window}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    return count <= limit
```

## Handling Expiration

Support URL expiration natively using Redis TTL:

```python
def create_short_url(r, long_url: str, ttl_seconds: int = None) -> str:
    short_code = generate_short_code(r)
    key = f"url:{short_code}"

    if ttl_seconds:
        r.setex(key, ttl_seconds, long_url)
    else:
        r.set(key, long_url)

    return short_code
```

Expired URLs automatically return NULL from `GET`, which you handle as a 404.

## Database and Redis Tier Responsibilities

| Concern           | Redis                    | Primary Database         |
|-------------------|--------------------------|--------------------------|
| URL redirect      | Hot cache (TTL-based)    | Persistent storage       |
| Click counting    | Real-time counters       | Aggregated analytics     |
| Rate limiting     | Per-user windows         | Not needed               |
| Key generation    | Atomic INCR counter      | Not needed               |
| Expiration        | TTL (volatile-ttl policy)| Soft deletes             |

## Redis Configuration for URL Shortener

```text
maxmemory 20gb
maxmemory-policy volatile-lru
hz 20
```

Use `volatile-lru` so only cached URL entries (with TTL) are evicted, while permanent counters without TTL are preserved.

## Summary

A URL shortener is a classic Redis use case that showcases multiple Redis features together: atomic counters for unique ID generation, string storage for URL mappings, sorted sets for time-series click analytics, and fixed-window counters for rate limiting. With Redis as the caching and counter layer, the system can handle 100K+ redirects per second with sub-10ms latency while the primary database handles persistence and complex queries.
