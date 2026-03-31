# How to Disable CloudEvents in Dapr Pub/Sub for Raw Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, CloudEvent, Raw Message, Integration

Description: Disable Dapr's automatic CloudEvents wrapping to publish and receive raw payloads for integration with non-Dapr consumers and legacy message brokers.

---

## Overview

By default, Dapr wraps all pub/sub messages in a CloudEvents 1.0 envelope. When interoperating with external systems (legacy apps, cloud provider SDKs, or other frameworks) that do not understand CloudEvents, you need raw message mode. Dapr supports this via the `rawPayload` metadata flag or a per-component configuration.

## With vs Without CloudEvents

```mermaid
graph LR
    subgraph CloudEvents mode (default)
        A1["Publish: {orderId:1}"] --> B1["Broker: {specversion,id,source,...,data:{orderId:1}}"]
    end
    subgraph Raw mode
        A2["Publish: {orderId:1}"] --> B2["Broker: {orderId:1}"]
    end
```

## Option 1: Per-Publish rawPayload Flag

Set the `rawPayload` metadata on a single publish call:

```bash
curl -X POST "http://localhost:3500/v1.0/publish/pubsub/orders?metadata.rawPayload=true" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "order-1", "total": 99.95}'
```

The broker receives the exact JSON body without any CloudEvents envelope.

## Option 2: SDK Publish with rawPayload

### Go

```go
meta := map[string]string{
    "rawPayload": "true",
}

err = client.PublishEvent(ctx, "pubsub", "orders",
    []byte(`{"orderId":"order-1","total":99.95}`),
    dapr.PublishEventWithMetadata(meta),
    dapr.PublishEventWithContentType("application/json"),
)
```

### Python

```python
from dapr.clients import DaprClient

async with DaprClient() as client:
    await client.publish_event(
        pubsub_name="pubsub",
        topic_name="orders",
        data=b'{"orderId":"order-1","total":99.95}',
        data_content_type="application/json",
        publish_metadata={"rawPayload": "true"},
    )
```

### TypeScript

```typescript
await client.pubsub.publish("pubsub", "orders",
  { orderId: "order-1", total: 99.95 },
  { metadata: { rawPayload: "true" } }
);
```

## Option 3: Component-Level Raw Payload

Configure the component to always use raw payloads (no CloudEvents wrapping for all messages):

```yaml
# components/pubsub-raw.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub-raw
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  # All publish/subscribe on this component bypass CloudEvents
```

For Kafka, use `rawPayload` in the component metadata:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-raw
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: kafka:9092
  - name: authType
    value: none
  - name: rawPayload
    value: "true"
```

## Subscribing to Raw Messages

When receiving raw messages, the payload is not wrapped in a CloudEvent. Configure the subscription to expect raw data:

### YAML Subscription with rawPayload

```yaml
# components/subscription-raw.yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-raw-subscription
spec:
  pubsubname: pubsub
  topic: orders
  route: /orders
  metadata:
    rawPayload: "true"
```

### Go Subscriber for Raw Messages

```go
func handleRawOrder(ctx context.Context, e *common.TopicEvent) (bool, error) {
    // e.RawData contains the raw bytes (no CloudEvent wrapper)
    var order map[string]interface{}
    if err := json.Unmarshal(e.RawData, &order); err != nil {
        return false, err
    }
    fmt.Printf("Raw order received: %v\n", order)
    return false, nil
}

s.AddTopicEventHandler(&common.Subscription{
    PubsubName: "pubsub",
    Topic:      "orders",
    Route:      "/orders",
    Metadata:   map[string]string{"rawPayload": "true"},
}, handleRawOrder)
```

### HTTP Endpoint for Raw Messages

```go
// Raw subscriber via HTTP
http.HandleFunc("/orders", func(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    // body is the raw payload, not a CloudEvent envelope
    var order map[string]interface{}
    json.Unmarshal(body, &order)
    fmt.Printf("Raw: %v\n", order)
    w.WriteHeader(http.StatusOK)
})
```

## Programmatic Subscription with rawPayload

```go
// Go SDK subscription with rawPayload
s.AddTopicEventHandler(&common.Subscription{
    PubsubName: "pubsub",
    Topic:      "legacy-orders",
    Route:      "/legacy-orders",
    Metadata:   map[string]string{"rawPayload": "true"},
}, handleLegacyOrder)
```

```python
# Python FastAPI
@dapr_app.subscribe(
    pubsub="pubsub",
    topic="legacy-orders",
    metadata={"rawPayload": "true"},
)
async def handle_legacy_order(event: dict):
    # event is the raw dict without CloudEvents fields
    print(f"Raw payload: {event}")
    return TopicEventResponse("success")
```

## Interoperability with External Producers

When an external system (non-Dapr) publishes to the same broker, it does not produce CloudEvents. Use raw mode on the subscriber to handle those messages:

```bash
# An external system publishes directly to Kafka
# No CloudEvent envelope
kafka-console-producer.sh --topic orders --broker kafka:9092
> {"orderId":"ext-1","source":"legacy-system"}

# Dapr subscriber with rawPayload=true will receive the plain JSON
```

## Summary

Dapr's default CloudEvents wrapping can be disabled per-publish using the `rawPayload=true` metadata flag, per-component via the `rawPayload` metadata field in the component YAML, or per-subscription via the subscription metadata. Raw mode is essential when integrating with external systems or legacy brokers that do not understand CloudEvents. Subscribers configured for raw mode receive the original payload bytes in `e.RawData` without CloudEvent fields.
