# How to Build a Shipment Tracking System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Shipment, Tracking

Description: Use Redis hashes, sorted sets, and pub/sub to build a real-time shipment tracking system with status history and live updates.

---

Customers expect live shipment updates. Polling a relational database for every status check is slow and expensive. Redis enables you to store shipment state, maintain history, and push real-time updates to customers - all from a single in-memory store.

## Storing Shipment State

Each shipment is stored as a hash with current status and metadata:

```python
import redis
import time
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_shipment(tracking_id: str, order_id: str, destination: str):
    pipe = r.pipeline()
    pipe.hset(f"shipment:{tracking_id}", mapping={
        "order_id": order_id,
        "destination": destination,
        "status": "created",
        "created_at": time.time(),
        "updated_at": time.time()
    })
    # Index by order ID
    pipe.set(f"order:shipment:{order_id}", tracking_id)
    # Add to active shipments sorted set (score = creation time)
    pipe.zadd("shipments:active", {tracking_id: time.time()})
    pipe.execute()
```

## Recording Status Updates

Each status change is appended to a list to maintain history:

```python
STATUS_ORDER = ["created", "picked_up", "in_transit", "out_for_delivery", "delivered"]

def update_shipment_status(tracking_id: str, status: str, location: str = ""):
    event = json.dumps({
        "status": status,
        "location": location,
        "timestamp": time.time()
    })

    pipe = r.pipeline()
    # Update current state
    pipe.hset(f"shipment:{tracking_id}", mapping={
        "status": status,
        "last_location": location,
        "updated_at": time.time()
    })
    # Append to history list (cap at 50 events)
    pipe.lpush(f"shipment:history:{tracking_id}", event)
    pipe.ltrim(f"shipment:history:{tracking_id}", 0, 49)
    # Publish real-time update
    pipe.publish(f"shipment:updates:{tracking_id}", event)

    # Move to delivered set if done
    if status == "delivered":
        pipe.zrem("shipments:active", tracking_id)
        pipe.zadd("shipments:delivered", {tracking_id: time.time()})
        # Expire delivered shipments after 90 days
        pipe.expire(f"shipment:{tracking_id}", 90 * 24 * 3600)
        pipe.expire(f"shipment:history:{tracking_id}", 90 * 24 * 3600)

    pipe.execute()
```

## Getting Shipment Details

```python
def get_shipment(tracking_id: str) -> dict:
    data = r.hgetall(f"shipment:{tracking_id}")
    if not data:
        return {}

    history_raw = r.lrange(f"shipment:history:{tracking_id}", 0, -1)
    history = [json.loads(h) for h in history_raw]

    return {
        "tracking_id": tracking_id,
        **data,
        "history": history
    }
```

## Subscribing to Live Updates

Frontend clients can receive real-time push notifications via pub/sub:

```python
def subscribe_to_shipment(tracking_id: str):
    pubsub = r.pubsub()
    pubsub.subscribe(f"shipment:updates:{tracking_id}")

    for message in pubsub.listen():
        if message["type"] == "message":
            update = json.loads(message["data"])
            print(f"Status: {update['status']} at {update['location']}")
            if update["status"] == "delivered":
                break

    pubsub.close()
```

## ETA Estimation

Store ETA as part of the shipment hash and update it as the package progresses:

```python
def set_eta(tracking_id: str, eta_timestamp: float):
    r.hset(f"shipment:{tracking_id}", "eta", eta_timestamp)
    # Notify subscribers
    r.publish(f"shipment:updates:{tracking_id}", json.dumps({
        "status": "eta_updated",
        "eta": eta_timestamp,
        "timestamp": time.time()
    }))
```

## Summary

Redis hashes give you fast reads of current shipment state, lists provide capped event history, and pub/sub delivers real-time status updates to subscribers. This combination handles high-throughput tracking operations while keeping storage lean with automatic expiration of delivered shipments.
