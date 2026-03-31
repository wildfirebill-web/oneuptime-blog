# How to Implement Session Revocation (Logout All Devices) with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Session, Security

Description: Implement global session revocation in Redis to instantly log out a user from all devices using token versioning and set-based tracking.

---

"Log out of all devices" is a critical security feature. Whether triggered by a password change, suspicious activity detection, or a user request, you need a way to instantly invalidate all active sessions for a user. Redis enables this with two clean approaches: tracking session IDs in a set, or using a version counter.

## Approach 1: Set-Based Revocation

Track all session IDs for a user in a Redis set and delete them all at once:

```python
import redis
import uuid
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

SESSION_TTL = 86400

def create_session(user_id: str, data: dict) -> str:
    session_id = str(uuid.uuid4())
    pipe = r.pipeline()
    pipe.setex(f"session:{session_id}", SESSION_TTL, json.dumps({"user_id": user_id, **data}))
    pipe.sadd(f"user:sessions:{user_id}", session_id)
    pipe.expire(f"user:sessions:{user_id}", SESSION_TTL)
    pipe.execute()
    return session_id

def logout_all_devices(user_id: str):
    session_ids = r.smembers(f"user:sessions:{user_id}")
    pipe = r.pipeline()
    for sid in session_ids:
        pipe.delete(f"session:{sid}")
    pipe.delete(f"user:sessions:{user_id}")
    pipe.execute()
    return len(session_ids)
```

## Approach 2: Token Version Counter (More Scalable)

Instead of tracking every session ID, store a version counter per user. Embed the current version in every session token, and invalidate all tokens by incrementing the counter:

```python
def get_current_version(user_id: str) -> int:
    version = r.get(f"user:version:{user_id}")
    return int(version) if version else 0

def create_session_versioned(user_id: str, data: dict) -> dict:
    session_id = str(uuid.uuid4())
    version = get_current_version(user_id)
    session_data = {"user_id": user_id, "version": version, **data}
    r.setex(f"session:{session_id}", SESSION_TTL, json.dumps(session_data))
    return {"session_id": session_id, "version": version}

def is_session_valid(session_id: str) -> bool:
    data_str = r.get(f"session:{session_id}")
    if not data_str:
        return False
    data = json.loads(data_str)
    current_version = get_current_version(data["user_id"])
    return data["version"] >= current_version

def revoke_all_sessions(user_id: str):
    # Increment version - all existing tokens are now invalid
    r.incr(f"user:version:{user_id}")
    r.expire(f"user:version:{user_id}", 86400 * 30)
```

## Handling Revocation in Middleware

```python
def authenticate(session_id: str) -> dict | None:
    if not is_session_valid(session_id):
        return None
    data = r.get(f"session:{session_id}")
    return json.loads(data) if data else None
```

## Post-Password-Change Revocation

Always revoke all sessions after a password change:

```python
def change_password(user_id: str, new_password_hash: str):
    update_password_in_db(user_id, new_password_hash)
    revoke_all_sessions(user_id)  # Force re-login on all devices
```

## Monitoring Revocation Activity

```bash
# Count active sessions for a user
redis-cli SCARD "user:sessions:user_123"

# Check current token version
redis-cli GET "user:version:user_123"
```

## Summary

Set-based revocation is simple and explicit - delete all tracked session keys at once. Version counter revocation is more scalable - a single INCR invalidates all sessions without enumerating them. Combine either approach with automatic revocation on password changes, and you have a robust "logout all devices" feature that eliminates unauthorized access immediately.
