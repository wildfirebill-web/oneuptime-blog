# How to Implement Per-User Rate Limiting with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Rate Limiting, Authentication

Description: Implement Redis per-user rate limiting to enforce individual request quotas for authenticated users independent of their IP address.

---

IP-based rate limiting is easy to bypass (shared NAT, VPNs) and unfair to users sharing an IP. Per-user rate limiting enforces quotas based on authenticated user identity, giving each user their own independent counter. This is the right approach for authenticated APIs.

## Per-User Rate Limit Key Design

Use the user's ID as part of the rate limit key so each user has an isolated counter:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def is_allowed(user_id: str, limit: int = 100, window_seconds: int = 60) -> dict:
    window = int(time.time() // window_seconds)
    key = f"ratelimit:user:{user_id}:{window}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window_seconds)
    pipe.ttl(key)
    results = pipe.execute()

    count = results[0]
    ttl = results[2]

    return {
        "allowed": count <= limit,
        "count": count,
        "limit": limit,
        "remaining": max(0, limit - count),
        "reset_in": ttl,
    }
```

## Tiered Limits by User Plan

Different users can have different limits based on their subscription tier:

```python
USER_TIER_LIMITS = {
    "free": {"limit": 100, "window": 3600},
    "pro": {"limit": 1000, "window": 3600},
    "enterprise": {"limit": 10000, "window": 3600},
}

def get_user_tier(user_id: str) -> str:
    tier = r.hget(f"user:{user_id}", "tier")
    return tier or "free"

def check_user_rate_limit(user_id: str) -> dict:
    tier = get_user_tier(user_id)
    config = USER_TIER_LIMITS[tier]
    return is_allowed(user_id, limit=config["limit"], window_seconds=config["window"])
```

## Flask Authentication Middleware

```python
from flask import Flask, request, jsonify, g
import jwt

app = Flask(__name__)
SECRET = "your-jwt-secret"

def get_current_user_id() -> str | None:
    auth = request.headers.get("Authorization", "")
    if not auth.startswith("Bearer "):
        return None
    try:
        token = auth[7:]
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
        return payload.get("user_id")
    except Exception:
        return None

@app.before_request
def per_user_rate_limit():
    user_id = get_current_user_id()
    if not user_id:
        return  # Not authenticated - IP rate limiting can handle this separately

    result = check_user_rate_limit(user_id)
    g.rate_limit = result

    if not result["allowed"]:
        return jsonify({
            "error": "Rate limit exceeded",
            "retry_after": result["reset_in"]
        }), 429
```

## Exempting Certain Users

Whitelist internal service accounts or admin users from rate limits:

```python
EXEMPT_USERS = {"service-account-monitor", "admin-bot"}

def is_allowed_with_exemption(user_id: str, limit: int, window_seconds: int) -> dict:
    if user_id in EXEMPT_USERS:
        return {"allowed": True, "limit": -1, "remaining": -1, "reset_in": 0}
    return is_allowed(user_id, limit, window_seconds)
```

## Viewing User-Specific Usage

```bash
# Check current request count for user
redis-cli GET "ratelimit:user:user_123:$(date +%s | awk '{print int($1/3600)}')"

# View all rate limit keys for a specific user
redis-cli --scan --pattern "ratelimit:user:user_123:*"
```

## Summary

Per-user rate limiting in Redis creates isolated counters for each authenticated user, making quotas fair and resistant to IP-based evasion. Adding tier-based limits lets you enforce different quotas for free vs. paid users with minimal overhead. Combining per-user limits with an exemption list for internal accounts keeps your infrastructure flexible without compromising protection.
