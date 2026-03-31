# How to Use Redis as a Service Mesh Data Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Microservice, Service Mesh, Architecture, Cache

Description: Use Redis as a shared data store for service mesh components to hold service routing tables, health state, and policy data across your microservices.

---

A service mesh controls traffic between microservices using sidecar proxies. The control plane needs a fast, consistent store for routing rules, circuit breaker state, and service health. Redis is a natural fit: sub-millisecond reads, pub/sub for configuration updates, and TTL-based expiry for stale state.

## Common Service Mesh Data in Redis

| Data Type | Redis Structure | Example Use |
|---|---|---|
| Service endpoints | Hash | `service:cart` -> `{ip, port, weight}` |
| Circuit breaker state | String + TTL | `cb:payment` -> `OPEN` |
| Routing rules | JSON / Hash | `route:checkout` -> `{canary: 10%}` |
| Health checks | Sorted Set | score = last_seen_timestamp |

## Storing Service Endpoints

```bash
HSET service:cart \
  host "10.0.1.42" \
  port "8080" \
  weight "100" \
  version "v2.1.0"

EXPIRE service:cart 30  # TTL: re-register or expire
```

Services re-register on each heartbeat, updating the TTL. If a service crashes, the key naturally expires.

## Circuit Breaker State

```python
import redis
import time

r = redis.Redis(host="redis", port=6379, decode_responses=True)

def set_circuit_open(service: str, duration_s: int = 30):
    r.setex(f"cb:{service}", duration_s, "OPEN")

def is_circuit_open(service: str) -> bool:
    return r.exists(f"cb:{service}") == 1

def record_failure(service: str, threshold: int = 5):
    key = f"failures:{service}"
    count = r.incr(key)
    r.expire(key, 60)  # reset window every 60s
    if count >= threshold:
        set_circuit_open(service, 30)
```

## Publishing Routing Rule Changes

When a route is updated, publish the change so all proxies reload without polling:

```python
# Control plane updates a route
r.hset("route:checkout", mapping={"canary_weight": "20", "stable_weight": "80"})
r.publish("route:updates", "checkout")
```

```python
# Sidecar proxy subscribes
pubsub = r.pubsub()
pubsub.subscribe("route:updates")

for message in pubsub.listen():
    if message["type"] == "message":
        service = message["data"]
        rules = r.hgetall(f"route:{service}")
        apply_routing_rules(service, rules)
```

## Health Tracking with Sorted Sets

Track last-seen timestamp as the score for quick staleness queries:

```bash
ZADD healthy_services $(date +%s) "cart"
ZADD healthy_services $(date +%s) "payment"

# Find services not seen in the last 60 seconds
ZRANGEBYSCORE healthy_services -inf $(($(date +%s) - 60))
```

## Service Discovery Lookup

```python
def get_service_endpoint(service: str) -> dict:
    endpoint = r.hgetall(f"service:{service}")
    if not endpoint:
        raise ServiceUnavailableError(f"{service} not registered")
    return endpoint
```

## Protecting Redis from Cascading Failures

Use read replicas for service discovery reads:

```python
read_redis = redis.Redis(host="redis-replica", port=6379)
write_redis = redis.Redis(host="redis-primary", port=6379)
```

Set short connection timeouts so a slow Redis does not cascade:

```python
r = redis.Redis(socket_connect_timeout=0.1, socket_timeout=0.1)
```

## Summary

Redis works well as a service mesh data store because its TTL-based keys naturally handle service deregistration, Pub/Sub pushes config changes to proxies without polling, and sorted sets enable efficient staleness queries. Separate read and write connections to replicas to protect the control plane under high read load.
