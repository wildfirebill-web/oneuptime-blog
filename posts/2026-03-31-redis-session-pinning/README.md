# How to Implement Session Pinning with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Session, Load Balancing

Description: Learn how to implement session pinning with Redis to ensure users are consistently routed to the same backend server instance.

---

Session pinning - also called sticky sessions - ensures that a user's requests are always routed to the same backend server. While stateless session storage in Redis eliminates the strict need for stickiness, some applications (WebSocket connections, streaming, or stateful in-memory operations) still benefit from it. Redis provides a clean way to implement software-level pinning without relying on load balancer configuration.

## What Is Session Pinning?

Without pinning, each request from the same user may land on a different backend server. With pinning, you store a server affinity mapping in Redis and use it to route requests consistently.

## Storing Server Affinity in Redis

When a session is created, assign it to a specific backend server:

```python
import redis
import uuid
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

SERVERS = ['backend-1', 'backend-2', 'backend-3']
PIN_TTL = 3600

def create_session_with_pin(user_id: str, assigned_server: str) -> str:
    session_id = str(uuid.uuid4())
    pipe = r.pipeline()

    # Store session data
    pipe.setex(
        f"session:{session_id}",
        PIN_TTL,
        json.dumps({"user_id": user_id})
    )

    # Store server affinity
    pipe.setex(f"session:pin:{session_id}", PIN_TTL, assigned_server)

    pipe.execute()
    return session_id

def get_pinned_server(session_id: str) -> str | None:
    return r.get(f"session:pin:{session_id}")
```

## Routing Requests Based on Pin

In your gateway or proxy layer, look up the pin before forwarding the request:

```python
import httpx

async def route_request(session_id: str, path: str, payload: dict) -> dict:
    server = get_pinned_server(session_id)

    if server is None:
        # No pin found - use default routing or create new session
        server = assign_server_round_robin()

    target_url = f"http://{server}{path}"
    async with httpx.AsyncClient() as client:
        response = await client.post(target_url, json=payload)
        return response.json()
```

## Auto-Reassignment on Server Failure

If the pinned server is unavailable, reassign the pin to a healthy server:

```python
def get_or_reassign_pin(session_id: str, healthy_servers: list) -> str:
    server = get_pinned_server(session_id)

    if server and server in healthy_servers:
        return server

    # Reassign to a healthy server
    new_server = pick_least_loaded(healthy_servers)
    r.setex(f"session:pin:{session_id}", PIN_TTL, new_server)
    return new_server

def pick_least_loaded(servers: list) -> str:
    loads = {s: int(r.get(f"server:load:{s}") or 0) for s in servers}
    return min(loads, key=loads.get)
```

## Updating Load Counters

Track request counts per server to enable load-aware pinning:

```python
def record_request(server: str):
    r.incr(f"server:load:{server}")
    r.expire(f"server:load:{server}", 60)  # Reset every minute
```

## Refreshing Pin TTL on Activity

Keep the pin alive as long as the user is active:

```python
def touch_pin(session_id: str):
    r.expire(f"session:pin:{session_id}", PIN_TTL)
    r.expire(f"session:{session_id}", PIN_TTL)
```

## Checking All Pins for a Server

```bash
# Find all sessions pinned to backend-2
redis-cli KEYS "session:pin:*" | xargs -I{} sh -c 'val=$(redis-cli GET {}); [ "$val" = "backend-2" ] && echo {}'
```

## Summary

Redis-based session pinning gives you server affinity without relying on load balancer sticky session configuration. By storing a session-to-server mapping in Redis and reading it on each request, you can route users consistently to the same backend while retaining the ability to reassign pins when servers go down. This keeps your architecture flexible and resilient.
