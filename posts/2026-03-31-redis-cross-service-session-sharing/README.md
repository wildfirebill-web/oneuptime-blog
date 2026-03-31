# How to Implement Cross-Service Session Sharing with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Session, Microservice, Authentication, Security

Description: Store and share user sessions in Redis so any microservice can authenticate requests without round-trips to a central auth service on every call.

---

In a microservices architecture, stateless HTTP requires every service to validate incoming tokens. Storing session data in Redis lets any service check session validity locally with a single Redis lookup, avoiding repeated calls to the auth service.

## Session Data Structure

Store sessions as Redis Hashes for efficient field-level access:

```bash
HSET session:abc123 \
  user_id "42" \
  username "alice" \
  roles "admin,editor" \
  created_at "1711843200" \
  last_seen "1711843800"

EXPIRE session:abc123 3600  # 1 hour TTL
```

## Creating a Session (Auth Service)

```python
import redis
import uuid
import time

r = redis.Redis(host="redis", port=6379, decode_responses=True)

def create_session(user_id: int, username: str, roles: list) -> str:
    session_id = str(uuid.uuid4())
    key = f"session:{session_id}"

    r.hset(key, mapping={
        "user_id": str(user_id),
        "username": username,
        "roles": ",".join(roles),
        "created_at": str(int(time.time()))
    })
    r.expire(key, 3600)
    return session_id
```

## Validating a Session (Any Service)

```python
def get_session(session_id: str) -> dict | None:
    key = f"session:{session_id}"
    data = r.hgetall(key)

    if not data:
        return None  # session expired or invalid

    # Sliding expiry: reset TTL on each use
    r.expire(key, 3600)

    return {
        "user_id": int(data["user_id"]),
        "username": data["username"],
        "roles": data["roles"].split(",")
    }
```

## FastAPI Middleware Example

```python
from fastapi import Request, HTTPException

async def session_middleware(request: Request, call_next):
    session_id = request.cookies.get("session_id") or \
                 request.headers.get("X-Session-ID")

    if not session_id:
        raise HTTPException(status_code=401, detail="No session")

    session = get_session(session_id)
    if not session:
        raise HTTPException(status_code=401, detail="Session expired")

    request.state.user = session
    return await call_next(request)
```

## Revoking a Session

Instant revocation is a key advantage over JWTs:

```python
def logout(session_id: str):
    r.delete(f"session:{session_id}")

def revoke_all_user_sessions(user_id: int):
    # Requires a secondary index: set of session IDs per user
    session_ids = r.smembers(f"user_sessions:{user_id}")
    if session_ids:
        pipeline = r.pipeline()
        for sid in session_ids:
            pipeline.delete(f"session:{sid}")
        pipeline.delete(f"user_sessions:{user_id}")
        pipeline.execute()
```

Maintain the secondary index when creating sessions:

```python
r.sadd(f"user_sessions:{user_id}", session_id)
r.expire(f"user_sessions:{user_id}", 86400)
```

## Security Considerations

- Use TLS between services and Redis (`requirepass` + TLS in redis.conf)
- Rotate session IDs after privilege escalation
- Set short TTLs for sensitive sessions (payment flows: 15 minutes)
- Never store passwords or secrets in the session hash

## Summary

Redis session sharing lets every microservice validate users with a single hash lookup instead of an auth service call on each request. Store sessions as Hashes with a TTL, use sliding expiry to keep active users logged in, and maintain a secondary index per user to enable instant revocation of all sessions. This pattern eliminates the main scalability bottleneck in centralized auth architectures.
