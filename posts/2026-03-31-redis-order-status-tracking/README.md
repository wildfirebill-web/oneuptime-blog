# How to Implement Order Status Tracking with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, E-commerce, Hash, Stream

Description: Track e-commerce order status changes with Redis Hashes for current state and Streams for full audit history - enable instant status lookups and real-time updates.

---

Order status tracking requires fast reads of the current state and a reliable audit trail of all transitions. Redis Hashes hold the current order state while Streams record every status change.

## Data Model

```text
order:{orderId}              -> Hash: status, user_id, total, updated_at
order:history:{orderId}     -> Stream: status transition events
orders:by_status:{status}   -> Set of order IDs in that status
orders:user:{userId}        -> Sorted Set: orderId -> created_at
```

## Creating an Order

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

ORDER_STATUSES = [
    "pending", "confirmed", "processing",
    "shipped", "delivered", "cancelled", "refunded"
]

def create_order(order_id, user_id, total):
    now = str(time.time())
    pipe = r.pipeline()
    pipe.hset(f"order:{order_id}", mapping={
        "id": order_id,
        "user_id": user_id,
        "total": str(total),
        "status": "pending",
        "created_at": now,
        "updated_at": now,
    })
    pipe.sadd("orders:by_status:pending", order_id)
    pipe.zadd(f"orders:user:{user_id}", {order_id: float(now)})
    pipe.xadd(f"order:history:{order_id}", {
        "status": "pending",
        "timestamp": now,
        "actor": "system",
    })
    pipe.execute()
```

## Updating Order Status

```python
def update_order_status(order_id, new_status, actor="system", note=""):
    if new_status not in ORDER_STATUSES:
        raise ValueError(f"Invalid status: {new_status}")

    current = r.hget(f"order:{order_id}", "status")
    if not current:
        return False

    now = str(time.time())
    pipe = r.pipeline()
    pipe.hset(f"order:{order_id}", mapping={
        "status": new_status,
        "updated_at": now,
    })
    pipe.srem(f"orders:by_status:{current}", order_id)
    pipe.sadd(f"orders:by_status:{new_status}", order_id)
    pipe.xadd(f"order:history:{order_id}", {
        "previous_status": current,
        "status": new_status,
        "timestamp": now,
        "actor": actor,
        "note": note,
    })
    pipe.execute()
    return True
```

## Getting Order Status

```python
def get_order(order_id):
    return r.hgetall(f"order:{order_id}")

def get_order_status(order_id):
    return r.hget(f"order:{order_id}", "status")

def get_order_history(order_id):
    entries = r.xrange(f"order:history:{order_id}", "-", "+")
    return [{"id": eid, **fields} for eid, fields in entries]
```

## Querying Orders by Status

```python
def get_orders_by_status(status):
    return r.smembers(f"orders:by_status:{status}")

def get_orders_by_status_count(status):
    return r.scard(f"orders:by_status:{status}")

def get_user_orders(user_id, limit=20):
    order_ids = r.zrevrange(f"orders:user:{user_id}", 0, limit - 1)
    pipe = r.pipeline()
    for oid in order_ids:
        pipe.hgetall(f"order:{oid}")
    return [o for o in pipe.execute() if o]
```

## Example Usage

```bash
# Create order
HSET order:1001 id 1001 user_id user:5 total 89.99 status pending
SADD orders:by_status:pending 1001
XADD order:history:1001 * status pending actor system

# Update to confirmed
HSET order:1001 status confirmed
SREM orders:by_status:pending 1001
SADD orders:by_status:confirmed 1001

# Get full history
XRANGE order:history:1001 - +
```

## Summary

Redis Hashes provide sub-millisecond order status reads while Streams maintain a full immutable audit trail of every status transition. Status index Sets enable efficient queries like "show all pending orders". This pattern supports both customer-facing status pages and internal operations dashboards with minimal latency.
