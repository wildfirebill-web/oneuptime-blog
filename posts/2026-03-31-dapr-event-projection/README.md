# How to Implement Event Projection with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Sourcing, Projection, Pub/Sub, Architecture

Description: Build event projections using Dapr pub/sub subscriptions to transform domain events into queryable read models updated in real time as events are published.

---

Event projections transform a stream of domain events into queryable read models optimized for specific views. In a Dapr-based event-sourced system, projections subscribe to pub/sub topics and update state stores as events arrive. This decouples write models (aggregates) from read models (projections).

## Projection Architecture

```
Domain Event Published
        |
        v
   Dapr Pub/Sub
        |
        v
  Projection Handler (subscriber)
        |
        v
   Read Model (Dapr State Store)
        |
        v
   Query Service (reads from state store)
```

## Defining a Projection

A projection subscribes to one or more event topics and maintains a denormalized read model.

```python
# projections/order_summary.py
from flask import Flask, request, jsonify
import requests
import json

app = Flask(__name__)
DAPR_URL = "http://localhost:3500"
READ_STORE = "readmodel-store"

@app.route("/dapr/subscribe")
def subscribe():
    return jsonify([
        {"pubsubname": "pubsub", "topic": "OrderCreated", "route": "/projections/order-created"},
        {"pubsubname": "pubsub", "topic": "OrderPaid", "route": "/projections/order-paid"},
        {"pubsubname": "pubsub", "topic": "OrderShipped", "route": "/projections/order-shipped"},
    ])

@app.route("/projections/order-created", methods=["POST"])
def handle_order_created():
    event = request.json.get("data", {})
    order_id = event.get("aggregateId")

    # Build the read model
    read_model = {
        "orderId": order_id,
        "customerId": event["payload"]["customerId"],
        "status": "pending",
        "total": event["payload"]["total"],
        "createdAt": event["occurredAt"]
    }

    # Persist to the read model store
    requests.post(
        f"{DAPR_URL}/v1.0/state/{READ_STORE}",
        json=[{"key": f"order-summary:{order_id}", "value": read_model}]
    )
    return jsonify({"status": "SUCCESS"})

@app.route("/projections/order-paid", methods=["POST"])
def handle_order_paid():
    event = request.json.get("data", {})
    order_id = event.get("aggregateId")

    # Load existing read model
    resp = requests.get(f"{DAPR_URL}/v1.0/state/{READ_STORE}/order-summary:{order_id}")
    if resp.status_code == 200 and resp.json():
        read_model = resp.json()
        read_model["status"] = "paid"
        read_model["paidAt"] = event["occurredAt"]

        requests.post(
            f"{DAPR_URL}/v1.0/state/{READ_STORE}",
            json=[{"key": f"order-summary:{order_id}", "value": read_model}]
        )
    return jsonify({"status": "SUCCESS"})
```

## Using Idempotency Keys

Events may be delivered more than once. Use the event sequence number as an idempotency key to prevent duplicate updates:

```python
@app.route("/projections/order-shipped", methods=["POST"])
def handle_order_shipped():
    event = request.json.get("data", {})
    order_id = event.get("aggregateId")
    seq = event.get("sequence")

    # Check if this event was already processed
    processed_key = f"processed:{order_id}:{seq}"
    resp = requests.get(f"{DAPR_URL}/v1.0/state/{READ_STORE}/{processed_key}")
    if resp.status_code == 200 and resp.json():
        return jsonify({"status": "SUCCESS"})  # Already processed

    # Process and mark as processed
    # ... update logic ...
    requests.post(f"{DAPR_URL}/v1.0/state/{READ_STORE}",
        json=[{"key": processed_key, "value": True}])

    return jsonify({"status": "SUCCESS"})
```

## Querying the Read Model

```python
@app.route("/orders/<order_id>/summary")
def get_order_summary(order_id):
    resp = requests.get(
        f"{DAPR_URL}/v1.0/state/{READ_STORE}/order-summary:{order_id}"
    )
    if resp.status_code == 204:
        return jsonify({"error": "not found"}), 404
    return resp.json()
```

## Rebuilding Projections

When a projection handler changes, wipe the read model store and replay all events:

```bash
# Flush projection read models
redis-cli FLUSHDB

# Trigger replay via an admin endpoint
curl -X POST http://localhost:8080/admin/replay-projections
```

## Summary

Dapr pub/sub subscriptions provide a clean mechanism for building event projections that update read models in real time. Idempotent handlers, combined with Dapr state management for read model persistence, give you reliable, queryable views that stay in sync with your domain events.
