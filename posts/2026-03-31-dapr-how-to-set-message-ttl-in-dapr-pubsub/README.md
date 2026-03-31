# How to Set Message TTL in Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, TTL, Message Expiry, Microservice

Description: Configure message time-to-live TTL in Dapr pub/sub to automatically expire stale messages and prevent outdated events from being processed by subscribers.

---

## Why Set Message TTL

Message TTL (Time-To-Live) automatically expires messages that have not been consumed within a specified time window. This prevents:

- Stale messages from being processed long after they are relevant
- Message backlogs from accumulating if consumers are slow or offline
- Outdated commands or events from being applied to changed system state
- Memory/disk exhaustion in message brokers from unbounded queues

## TTL Support by Message Broker

Dapr supports message TTL for these pub/sub components:

| Component | TTL Support |
|-----------|------------|
| Redis Streams | Yes |
| RabbitMQ | Yes |
| Azure Service Bus | Yes |
| Apache Kafka | Yes (via log retention) |
| NATS | Yes |
| AWS SQS | Yes |

## Setting TTL at the Component Level

Configure a default TTL for all messages on a pub/sub component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-master:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis
      key: redis-password
  - name: ttlInSeconds
    value: "3600"
```

This sets a 1-hour TTL for all messages published through this component. Messages not consumed within 1 hour are automatically deleted.

## Setting TTL Per Message When Publishing

Override the component TTL for specific messages using the `ttlInSeconds` metadata:

```bash
# Publish a message with 5-minute TTL
curl -X POST "http://localhost:3500/v1.0/publish/pubsub/orders?metadata.ttlInSeconds=300" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123", "total": 99.99, "urgency": "high"}'
```

```bash
# Publish a long-lived message with 24-hour TTL
curl -X POST "http://localhost:3500/v1.0/publish/pubsub/notifications?metadata.ttlInSeconds=86400" \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-123", "type": "daily-digest"}'
```

## Per-Message TTL from Python

```python
import requests
import json

DAPR_HTTP_PORT = 3500

def publish_with_ttl(topic: str, data: dict, ttl_seconds: int):
    """Publish a message with a specific TTL"""
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/publish/pubsub/{topic}"
    params = {"metadata.ttlInSeconds": str(ttl_seconds)}
    
    response = requests.post(
        url,
        params=params,
        headers={"Content-Type": "application/json"},
        data=json.dumps(data)
    )
    
    if response.status_code == 204:
        print(f"Published to {topic} with {ttl_seconds}s TTL")
    else:
        print(f"Failed: {response.status_code}")

# Time-sensitive order (5 minute TTL - must process quickly)
publish_with_ttl("flash-orders", {
    "orderId": "order-123",
    "discountCode": "FLASH50",
    "total": 49.99
}, ttl_seconds=300)

# Password reset notification (1 hour TTL)
publish_with_ttl("notifications", {
    "userId": "user-456",
    "type": "password-reset",
    "token": "secure-token-abc"
}, ttl_seconds=3600)

# Background sync task (24 hour TTL)
publish_with_ttl("sync-tasks", {
    "taskId": "sync-789",
    "resourceType": "user-profile"
}, ttl_seconds=86400)
```

## Using the Dapr SDK with TTL

```python
from dapr.clients import DaprClient
import json

with DaprClient() as client:
    # Short-lived price update message (60 second TTL)
    price_update = {
        "productId": "prod-001",
        "newPrice": 29.99,
        "effectiveAt": "2026-03-31T10:00:00Z"
    }
    
    client.publish_event(
        pubsub_name="pubsub",
        topic_name="price-updates",
        data=json.dumps(price_update),
        data_content_type="application/json",
        publish_metadata={"ttlInSeconds": "60"}
    )
    
    print("Price update published with 60s TTL")
```

## Node.js with Per-Message TTL

```javascript
const axios = require('axios');

const DAPR_HTTP_PORT = process.env.DAPR_HTTP_PORT || 3500;

async function publishWithTTL(topic, data, ttlSeconds) {
  const url = `http://localhost:${DAPR_HTTP_PORT}/v1.0/publish/pubsub/${topic}`;
  
  await axios.post(url, data, {
    params: { 'metadata.ttlInSeconds': ttlSeconds.toString() },
    headers: { 'Content-Type': 'application/json' }
  });
  
  console.log(`Published to ${topic} with TTL ${ttlSeconds}s`);
}

// Usage examples
async function main() {
  // Session token notification - expires in 15 minutes
  await publishWithTTL('auth-events', {
    userId: 'user-123',
    sessionId: 'session-abc',
    action: 'login'
  }, 900);
  
  // Shipping update - valid for 24 hours
  await publishWithTTL('shipping-updates', {
    orderId: 'order-456',
    status: 'in-transit',
    estimatedDelivery: '2026-04-01'
  }, 86400);
}

main().catch(console.error);
```

## Designing TTL Values for Common Patterns

| Message Type | Recommended TTL | Reason |
|-------------|----------------|--------|
| Real-time price updates | 30-60 seconds | Prices change frequently |
| Order confirmation | 1-24 hours | Customer expects timely processing |
| Password reset links | 15-60 minutes | Security requirement |
| Daily digest notifications | 24 hours | Must reach user within the day |
| Background sync tasks | 24-72 hours | Can tolerate delay |
| Flash sale events | 5-15 minutes | Sale ends soon |

## Handling Expired Messages

When consumers are offline and messages expire, design your system to handle gaps:

```python
@app.route('/orders/process', methods=['POST'])
def process_order():
    event = request.get_json()
    order = event.get('data', {})
    
    # Check if the order is still valid by looking at order creation time
    from datetime import datetime, timezone, timedelta
    
    created_at_str = order.get('createdAt')
    if created_at_str:
        created_at = datetime.fromisoformat(created_at_str)
        now = datetime.now(timezone.utc)
        
        # Double-check expiry even if TTL didn't catch it
        if (now - created_at) > timedelta(hours=1):
            print(f"Order {order.get('orderId')} is too old - dropping")
            return jsonify({"status": "DROP"})
    
    # Process the fresh order
    return jsonify({"status": "SUCCESS"})
```

## Summary

Dapr message TTL prevents stale messages from being processed by automatically expiring them after a specified duration. Set TTL at the component level for a default across all messages, or override per-message using the `metadata.ttlInSeconds` query parameter when publishing. Design TTL values based on how quickly messages become irrelevant in your business domain, and implement subscriber-side validation to handle edge cases where messages arrive after their expected freshness window.
