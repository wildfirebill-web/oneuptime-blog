# How to Design a Session Store Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Session Store, Interview, Authentication

Description: Design a distributed session store with Redis covering session data modeling, TTL management, sticky vs stateless sessions, and horizontal scaling.

---

## Requirements Clarification

**Functional:**
- Store session data after user login
- Retrieve session data on each request
- Invalidate sessions on logout or expiry
- Support session extension on activity

**Non-functional:**
- 50M active users
- 100K requests/sec peak
- Session data under 10KB per session
- P99 session read latency under 5ms
- Sessions expire after 30 minutes of inactivity

## Scale Estimation

```text
Active sessions: 50M
Session size: 2KB average
Total memory: 50M * 2KB = ~100GB
Read QPS: 100,000 req/sec
Write QPS (new logins): ~5,000 req/sec
Session TTL refresh QPS: ~100,000 req/sec (on every request)
```

## Session Data Model

Use Redis Hashes to store structured session data efficiently:

```bash
HSET session:abc123xyz \
  user_id "1001" \
  email "alice@example.com" \
  role "admin" \
  ip "192.168.1.1" \
  created_at "1711900000" \
  last_seen "1711903600"

EXPIRE session:abc123xyz 1800
```

Hash encoding keeps small sessions (under 128 fields, each under 64 bytes) in compact listpack encoding, reducing memory to ~150 bytes per session.

Alternatively, use JSON strings for flexibility:

```bash
SET session:abc123xyz '{"userId":1001,"role":"admin","email":"alice@example.com"}' EX 1800
```

## Session ID Generation

Session IDs must be:
1. **Unpredictable**: use cryptographic random bytes
2. **Long enough**: 128-bit (16 bytes) = 32 hex chars or 22 base64url chars

```python
import secrets
import redis

r = redis.Redis(host='localhost', port=6379)

def generate_session_id() -> str:
    return secrets.token_urlsafe(16)  # 22-char URL-safe base64 string

def create_session(user_id: int, role: str, email: str, ip: str) -> str:
    session_id = generate_session_id()
    session_key = f"session:{session_id}"

    r.hset(session_key, mapping={
        "user_id": user_id,
        "role": role,
        "email": email,
        "ip": ip,
        "created_at": int(time.time()),
        "last_seen": int(time.time())
    })
    r.expire(session_key, 1800)  # 30 minutes

    return session_id
```

## Session Retrieval and TTL Refresh

On every authenticated request, retrieve the session and extend the TTL:

```python
import time

def get_session(session_id: str) -> dict | None:
    session_key = f"session:{session_id}"

    # Get all fields atomically with TTL refresh
    pipe = r.pipeline()
    pipe.hgetall(session_key)
    pipe.expire(session_key, 1800)
    results = pipe.execute()

    session_data = results[0]
    if not session_data:
        return None

    return {k.decode(): v.decode() for k, v in session_data.items()}
```

Using a pipeline makes the HGETALL + EXPIRE atomic from the client's perspective (two round trips are avoided).

For maximum atomicity, use a Lua script:

```lua
local key = KEYS[1]
local ttl = ARGV[1]
local data = redis.call('HGETALL', key)
if #data == 0 then
    return nil
end
redis.call('EXPIRE', key, ttl)
return data
```

## Session Invalidation

Logout invalidates the session immediately:

```python
def logout(session_id: str) -> None:
    r.delete(f"session:{session_id}")
```

For admin-initiated forced logout (e.g., after a password change), maintain a set of active sessions per user:

```python
def create_session(user_id: int, ...) -> str:
    session_id = generate_session_id()
    session_key = f"session:{session_id}"
    user_sessions_key = f"user_sessions:{user_id}"

    pipe = r.pipeline()
    pipe.hset(session_key, mapping={...})
    pipe.expire(session_key, 1800)
    pipe.sadd(user_sessions_key, session_id)
    pipe.expire(user_sessions_key, 86400)  # clean up user session set after 1 day
    pipe.execute()

    return session_id

def logout_all_sessions(user_id: int) -> None:
    user_sessions_key = f"user_sessions:{user_id}"
    session_ids = r.smembers(user_sessions_key)

    pipe = r.pipeline()
    for session_id in session_ids:
        pipe.delete(f"session:{session_id.decode()}")
    pipe.delete(user_sessions_key)
    pipe.execute()
```

## Sticky Sessions vs Stateless Sessions

**Sticky sessions (server affinity):**
Session data is stored in-process. The load balancer routes each user to the same server.
- Problem: server restarts lose all sessions; poor for horizontal scaling.

**Stateless sessions (Redis-backed):**
Session data is in Redis. Any server can handle any request.

```text
[User] --> [Load Balancer] --> [Server 1]  \
                          --> [Server 2]   -> [Redis Cluster]
                          --> [Server 3]  /
```

All servers share the same Redis, so session data is available regardless of which server handles the request.

## Middleware Example (Node.js/Express)

```javascript
const redis = require('redis');
const client = redis.createClient({ url: 'redis://localhost:6379' });

async function sessionMiddleware(req, res, next) {
  const sessionId = req.cookies['session_id'];
  if (!sessionId) {
    req.session = null;
    return next();
  }

  const key = `session:${sessionId}`;
  const data = await client.hGetAll(key);

  if (!data || Object.keys(data).length === 0) {
    res.clearCookie('session_id');
    req.session = null;
    return next();
  }

  // Extend TTL on activity
  await client.expire(key, 1800);

  req.session = data;
  next();
}
```

## Scaling the Session Store

100GB of session data exceeds a single Redis instance. Use Redis Cluster:

```text
Redis Cluster with 3 primary + 3 replica nodes:
  Node 1: slots 0-5460       (33GB)
  Node 2: slots 5461-10922   (33GB)
  Node 3: slots 10923-16383  (34GB)
```

Session keys distribute automatically across slots based on the hash of `session:{id}`.

Alternatively, use a consistent hash ring if running standalone nodes:

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    startup_nodes=[
        {"host": "redis-1", "port": "6379"},
        {"host": "redis-2", "port": "6379"},
        {"host": "redis-3", "port": "6379"},
    ]
)
```

## Redis Configuration for Session Store

```text
maxmemory 120gb
maxmemory-policy volatile-lru
hz 20
```

Use `volatile-lru` to evict the least recently used sessions when memory is tight, preserving keys without TTL (admin data, counters).

## Summary

Designing a Redis session store requires careful attention to session ID randomness, compact data modeling with hashes, atomic TTL refresh on access, and efficient multi-session invalidation via per-user sets. Redis is the ideal session store for horizontally-scaled applications because it decouples session state from individual servers, enables sub-millisecond lookups, and automatically expires stale sessions via TTL. Scaling to 50M concurrent sessions requires Redis Cluster with sufficient RAM provisioned for peak usage plus headroom.
