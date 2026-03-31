# How to Use Dapr Pub/Sub with Event-Driven Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Event-Driven Architecture, Microservices, Kubernetes, Events

Description: Learn how to design and implement event-driven microservices using Dapr pub/sub, including event sourcing patterns, saga orchestration, and service decoupling.

---

## Event-Driven Architecture with Dapr

Event-Driven Architecture (EDA) is a design pattern where services communicate through events rather than direct calls. Dapr's pub/sub building block is the primary tool for implementing EDA, providing:

- **Decoupling**: Publishers don't know who consumes their events
- **Scalability**: Multiple consumers can independently scale
- **Resilience**: Message broker buffers events during consumer downtime
- **Auditability**: All state changes are captured as events

## Core EDA Patterns with Dapr Pub/Sub

### Pattern 1: Domain Events

Services emit events when significant business actions occur:

```python
# Order service emits events after state changes
from flask import Flask, request, jsonify
import requests
import json

app = Flask(__name__)
DAPR_PORT = 3500

def publish_event(topic: str, event_type: str, data: dict):
    """Publish a domain event"""
    response = requests.post(
        f"http://localhost:{DAPR_PORT}/v1.0/publish/pubsub/{topic}",
        params={"metadata.cloudevent.type": event_type},
        headers={"Content-Type": "application/json"},
        data=json.dumps(data)
    )
    return response.status_code == 204

@app.route('/orders', methods=['POST'])
def create_order():
    order_data = request.get_json()
    
    # Save the order (simplified)
    order_id = save_order(order_data)
    
    # Emit domain event - other services react to this
    publish_event(
        topic="order-events",
        event_type="com.mycompany.orders.created",
        data={
            "orderId": order_id,
            "customerId": order_data['customerId'],
            "items": order_data['items'],
            "total": order_data['total'],
            "status": "created"
        }
    )
    
    return jsonify({"orderId": order_id, "status": "created"})

def save_order(order_data: dict) -> str:
    # Database logic here
    return "order-" + "12345"
```

### Pattern 2: Event-Driven Saga

Coordinate distributed transactions using events:

```text
Order Service --> "order.created" --> Payment Service
                                           |
                                    "payment.completed"
                                           |
                                    Inventory Service
                                           |
                                    "inventory.reserved"
                                           |
                                    Shipping Service
                                           |
                                    "shipping.scheduled"
```

Each service subscribes to events from the previous step:

```python
# Payment service subscribes to order events
@app.route('/dapr/subscribe', methods=['GET'])
def subscribe():
    return jsonify([{
        'pubsubname': 'pubsub',
        'topic': 'order-events',
        'route': '/payments/handle-order-event'
    }])

@app.route('/payments/handle-order-event', methods=['POST'])
def handle_order_event():
    event = request.get_json()
    event_type = event.get('type')
    order_data = event.get('data', {})
    
    # Only process order.created events
    if event_type != 'com.mycompany.orders.created':
        return jsonify({"status": "SUCCESS"})
    
    order_id = order_data['orderId']
    total = order_data['total']
    
    try:
        # Process payment
        payment_id = charge_customer(order_data['customerId'], total)
        
        # Emit payment completed event
        publish_event(
            topic="payment-events",
            event_type="com.mycompany.payments.completed",
            data={
                "orderId": order_id,
                "paymentId": payment_id,
                "amount": total,
                "status": "completed"
            }
        )
        
        return jsonify({"status": "SUCCESS"})
    
    except PaymentFailedException as e:
        # Emit compensating event to cancel the order
        publish_event(
            topic="order-events",
            event_type="com.mycompany.orders.payment_failed",
            data={
                "orderId": order_id,
                "reason": str(e)
            }
        )
        
        return jsonify({"status": "SUCCESS"})

def charge_customer(customer_id: str, amount: float) -> str:
    # Payment gateway logic
    return "payment-" + "abc123"
```

### Pattern 3: CQRS with Event Sourcing

Separate read and write models using events:

```python
# Write side - command handler emits events
@app.route('/products/update-price', methods=['POST'])
def update_product_price():
    data = request.get_json()
    product_id = data['productId']
    new_price = data['newPrice']
    
    # Record the event (event sourcing)
    event_id = append_to_event_store({
        "type": "product.price_updated",
        "productId": product_id,
        "oldPrice": get_current_price(product_id),
        "newPrice": new_price,
        "updatedBy": data.get('userId')
    })
    
    # Publish for read model updates
    publish_event(
        topic="product-events",
        event_type="com.mycompany.products.price_updated",
        data={
            "eventId": event_id,
            "productId": product_id,
            "newPrice": new_price
        }
    )
    
    return jsonify({"status": "updated"})
```

```python
# Read side - updates search index and cache from events
@app.route('/dapr/subscribe', methods=['GET'])
def subscribe():
    return jsonify([{
        'pubsubname': 'pubsub',
        'topic': 'product-events',
        'route': '/read-model/handle'
    }])

@app.route('/read-model/handle', methods=['POST'])
def update_read_model():
    event = request.get_json()
    event_type = event.get('type')
    data = event.get('data', {})
    
    if event_type == 'com.mycompany.products.price_updated':
        product_id = data['productId']
        new_price = data['newPrice']
        
        # Update search index
        update_elasticsearch_price(product_id, new_price)
        
        # Update cache
        invalidate_product_cache(product_id)
        
        print(f"Read model updated for product {product_id}")
    
    return jsonify({"status": "SUCCESS"})
```

## Multi-Consumer Fan-Out Pattern

Multiple services can subscribe to the same topic independently:

```text
"order-events" topic
       |
       |---- inventory-service (reserves stock)
       |---- notification-service (sends confirmation email)
       |---- analytics-service (updates sales metrics)
       |---- fulfillment-service (schedules warehouse pick)
```

Each service subscribes with its own unique `app-id`, receiving its own copy of each message:

```yaml
# Inventory service deployment
metadata:
  annotations:
    dapr.io/app-id: "inventory-service"

# Notification service deployment
metadata:
  annotations:
    dapr.io/app-id: "notification-service"
```

## Event Schema Versioning

Handle schema evolution by including version information:

```python
def publish_order_event_v2(order: dict):
    """Publish order event with version for backward compatibility"""
    publish_event(
        topic="order-events",
        event_type="com.mycompany.orders.created",
        data={
            "schemaVersion": "2.0",
            "orderId": order['id'],
            "customerId": order['customerId'],
            "items": order['items'],
            "total": order['total'],
            # New field in v2
            "shippingAddress": order.get('shippingAddress')
        }
    )

# Subscriber handles multiple schema versions
@app.route('/inventory/handle-order', methods=['POST'])
def handle_order():
    event = request.get_json()
    data = event.get('data', {})
    
    schema_version = data.get('schemaVersion', '1.0')
    
    if schema_version == '1.0':
        # Handle legacy format
        items = data.get('items', [])
    elif schema_version == '2.0':
        # Handle new format with additional fields
        items = data.get('items', [])
        shipping = data.get('shippingAddress')
    
    reserve_inventory(data['orderId'], items)
    return jsonify({"status": "SUCCESS"})
```

## Summary

Dapr pub/sub enables event-driven microservices through decoupled domain event publishing, event-driven sagas for distributed transactions, and CQRS read model projections. Design services to emit domain events on state changes, subscribe to events from upstream services, and emit compensating events on failure for saga rollback. Use consistent event type naming (e.g., `com.company.domain.action`) and include schema version fields to support backward-compatible event evolution.
