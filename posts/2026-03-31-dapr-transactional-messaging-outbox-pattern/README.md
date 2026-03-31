# How to Implement Transactional Messaging with Dapr Outbox Pattern

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Outbox Pattern, Transactional Messaging, Pub/Sub, State Management

Description: Learn how to use the Dapr outbox pattern to atomically save state and publish events, ensuring consistency between your database and message broker.

---

## The Dual-Write Problem

When a service needs to both save data and publish an event, doing them separately creates a consistency problem. If the service saves data but crashes before publishing, the event is lost. If it publishes first and then the database write fails, an event was sent for something that never happened.

The outbox pattern solves this: save both the data and the outgoing message in the same transaction. A separate process (or Dapr itself) reliably delivers the message afterward.

## Dapr's Built-In Outbox Support

Dapr supports the outbox pattern natively starting from version 1.12. Enable it in the state store component configuration:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.postgresql
  version: v1
  metadata:
  - name: connectionString
    value: "host=localhost user=postgres password=secret dbname=orders port=5432"
  - name: outboxPublishPubsub
    value: "messagebus"
  - name: outboxPublishTopic
    value: "outbox-events"
  - name: outboxDiscardWhenMissingState
    value: "false"
```

Also configure the pub/sub component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
  namespace: default
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://guest:guest@localhost:5672"
  - name: durable
    value: "true"
```

## Publishing an Event via the Outbox

With the outbox configured, use a Dapr transaction to save state and publish an event atomically:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    dapr "github.com/dapr/go-sdk/client"
)

type Order struct {
    ID         string  `json:"id"`
    CustomerID string  `json:"customer_id"`
    Total      float64 `json:"total"`
    Status     string  `json:"status"`
}

type OrderPlacedEvent struct {
    OrderID    string  `json:"order_id"`
    CustomerID string  `json:"customer_id"`
    Total      float64 `json:"total"`
}

func placeOrder(ctx context.Context, client dapr.Client, order Order) error {
    orderData, _ := json.Marshal(order)
    eventData, _ := json.Marshal(OrderPlacedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Total:      order.Total,
    })

    ops := []*dapr.StateOperation{
        // Save the order to state
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "order:" + order.ID,
                Value: orderData,
            },
        },
        // This is the outbox message - Dapr will publish it atomically
        {
            Type: dapr.StateOperationTypeUpsert,
            Item: &dapr.SetStateItem{
                Key:   "outbox:order-placed:" + order.ID,
                Value: eventData,
                Metadata: map[string]string{
                    "outbox.topic":       "order-events",
                    "outbox.pubsubname":  "messagebus",
                },
            },
        },
    }

    err := client.ExecuteStateTransaction(ctx, "statestore", nil, ops)
    if err != nil {
        return fmt.Errorf("transaction failed: %w", err)
    }

    fmt.Printf("Order %s saved and event queued via outbox\n", order.ID)
    return nil
}
```

## Subscribing to Outbox Events

Subscribe to the outbox topic to consume published events:

```python
from fastapi import FastAPI
from dapr.ext.fastapi import DaprApp

app = FastAPI()
dapr_app = DaprApp(app)

@dapr_app.subscribe(pubsub="messagebus", topic="order-events")
async def handle_order_event(event_data: dict):
    event_type = event_data.get("type", "unknown")
    order_id = event_data.get("data", {}).get("order_id")

    print(f"Received outbox event: {event_type} for order {order_id}")

    if event_type == "order.placed":
        await send_order_confirmation_email(event_data["data"])
    elif event_type == "order.shipped":
        await notify_customer_of_shipment(event_data["data"])

    return {"success": True}
```

## Manual Outbox Implementation (Without Native Support)

For state stores that do not support the native outbox, implement it manually:

```python
import json
import uuid
from datetime import datetime
from dapr.clients import DaprClient

def save_with_outbox(entity_key: str, entity_data: dict, event_type: str, event_payload: dict):
    """Save entity and create outbox record in a single transaction."""
    outbox_id = str(uuid.uuid4())

    outbox_record = {
        "id":         outbox_id,
        "event_type": event_type,
        "payload":    event_payload,
        "created_at": datetime.utcnow().isoformat(),
        "published":  False
    }

    with DaprClient() as client:
        ops = [
            {
                "operation": "upsert",
                "request": {
                    "key": entity_key,
                    "value": json.dumps(entity_data)
                }
            },
            {
                "operation": "upsert",
                "request": {
                    "key": f"outbox:{outbox_id}",
                    "value": json.dumps(outbox_record)
                }
            }
        ]
        client.execute_state_transaction("statestore", ops)
        return outbox_id

def outbox_relay_loop():
    """Periodically scan for unpublished outbox records and publish them."""
    with DaprClient() as client:
        while True:
            # Query outbox records (implement query based on your state store)
            unpublished = get_unpublished_outbox_records(client)

            for record in unpublished:
                try:
                    client.publish_event(
                        pubsub_name="messagebus",
                        topic_name=record["event_type"],
                        data=json.dumps(record["payload"])
                    )
                    # Mark as published
                    record["published"] = True
                    client.save_state("statestore", f"outbox:{record['id']}",
                                      json.dumps(record))
                    print(f"Published outbox event {record['id']}")
                except Exception as e:
                    print(f"Failed to publish {record['id']}: {e}")

            time.sleep(1)
```

## Testing the Outbox Pattern

```bash
# Start the service
dapr run --app-id order-service \
         --app-port 5001 \
         --dapr-http-port 3500 \
         --resources-path ./components \
         -- python main.py

# Place an order
curl -X POST http://localhost:5001/orders \
  -H "Content-Type: application/json" \
  -d '{
    "id": "ORD-2001",
    "customer_id": "CUST-42",
    "total": 79.99
  }'

# Verify the event was published
# (Check your subscriber or message broker console)
```

## Summary

The outbox pattern eliminates the dual-write problem in distributed systems by ensuring state changes and event publications happen atomically. Dapr's native outbox support makes this straightforward - configure the outbox topic in your state store component, and Dapr handles the reliable delivery. For state stores without native support, a manual outbox table with a relay loop achieves the same consistency guarantee, ensuring no events are lost even when services crash mid-operation.
