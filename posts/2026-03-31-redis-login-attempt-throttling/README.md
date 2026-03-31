# How to Implement Login Attempt Throttling with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Security, Authentication

Description: Throttle login attempts with Redis to prevent brute force attacks by tracking failed attempts and imposing progressive delays or lockouts.

---

Brute force attacks on login endpoints are one of the most common attack vectors. Redis makes it easy to track failed login attempts and implement progressive lockouts - slowing attackers down or locking out accounts after too many failures, while minimizing friction for legitimate users.

## Strategy: Track Per-Account and Per-IP

Track both the account being targeted (to protect individual accounts) and the IP (to block distributed attacks):

```python
import redis
import time
import math

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

MAX_ATTEMPTS = 5
LOCKOUT_DURATION = 900  # 15 minutes

def record_failed_attempt(identifier: str) -> int:
    key = f"login:failures:{identifier}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, LOCKOUT_DURATION)
    return count

def get_failure_count(identifier: str) -> int:
    return int(r.get(f"login:failures:{identifier}") or 0)

def is_locked_out(identifier: str) -> bool:
    return get_failure_count(identifier) >= MAX_ATTEMPTS

def reset_failures(identifier: str):
    r.delete(f"login:failures:{identifier}")
```

## Login Handler with Throttling

```python
def login(username: str, password: str, ip_address: str) -> dict:
    user_key = f"account:{username}"
    ip_key = f"ip:{ip_address}"

    # Check account and IP lockouts
    if is_locked_out(user_key):
        ttl = r.ttl(f"login:failures:{user_key}")
        return {"success": False, "error": "Account locked", "retry_after": ttl}

    if is_locked_out(ip_key):
        ttl = r.ttl(f"login:failures:{ip_key}")
        return {"success": False, "error": "Too many attempts from this IP", "retry_after": ttl}

    # Verify credentials
    if not verify_password(username, password):
        user_failures = record_failed_attempt(user_key)
        record_failed_attempt(ip_key)
        remaining = max(0, MAX_ATTEMPTS - user_failures)
        return {"success": False, "error": "Invalid credentials", "attempts_remaining": remaining}

    # Success - clear failure counters
    reset_failures(user_key)
    reset_failures(ip_key)
    return {"success": True, "token": create_session_token(username)}
```

## Progressive Delay (Exponential Backoff)

Instead of hard lockouts, impose increasing delays per failed attempt:

```python
def get_delay_seconds(identifier: str) -> float:
    count = get_failure_count(identifier)
    if count < 2:
        return 0
    return min(30, math.pow(2, count - 1))  # 2, 4, 8, 16, 30 seconds

def is_in_delay_period(identifier: str) -> tuple[bool, float]:
    delay_key = f"login:delay:{identifier}"
    delay_until = r.get(delay_key)
    if delay_until:
        remaining = float(delay_until) - time.time()
        if remaining > 0:
            return True, remaining
    return False, 0.0

def apply_delay(identifier: str):
    delay = get_delay_seconds(identifier)
    if delay > 0:
        r.setex(f"login:delay:{identifier}", int(delay) + 1, time.time() + delay)
```

## CAPTCHA Trigger after N Failures

```python
def should_require_captcha(identifier: str, threshold: int = 3) -> bool:
    return get_failure_count(identifier) >= threshold
```

## Monitoring Login Failures

```bash
# View accounts with active lockouts
redis-cli --scan --pattern "login:failures:account:*"

# Check failure count for a specific account
redis-cli GET "login:failures:account:john@example.com"

# Clear a lockout manually (for support requests)
redis-cli DEL "login:failures:account:john@example.com"
```

## Summary

Redis-based login throttling protects against brute force attacks with minimal overhead. Tracking failures per account AND per IP addresses both targeted credential attacks and distributed spray attacks. Progressive delays add friction without permanent lockouts, while hard lockouts after N attempts protect high-value accounts. Always provide a manual admin override to clear lockouts for legitimate users who have been locked out.
