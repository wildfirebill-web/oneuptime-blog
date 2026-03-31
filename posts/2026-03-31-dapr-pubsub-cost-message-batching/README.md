# How to Optimize Dapr Pub/Sub Costs with Message Batching

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Cost Optimization, Message Broker, Performance

Description: Learn how to configure Dapr pub/sub message batching to reduce broker API calls, lower cloud messaging costs, and improve throughput for high-volume services.

---

## Why Message Batching Reduces Costs

Cloud-hosted message brokers like Azure Service Bus, AWS SQS, and Google Pub/Sub charge per API call or per message operation. Publishing or consuming messages one-by-one can multiply your bill significantly under high load. Dapr's pub/sub batching allows multiple messages to be grouped into a single broker operation, reducing API calls and associated costs.

## Configuring Bulk Publish in Dapr

Dapr 1.10+ supports bulk publish, which sends multiple messages in a single HTTP call:

```go
package main

import (
    "context"
    dapr "github.com/dapr/go-sdk/client"
)

func main() {
    client, _ := dapr.NewClient()
    defer client.Close()

    messages := []dapr.BulkPublishRequestEntry{
        {EntryID: "1", Event: []byte(`{"orderId": "101"}`), ContentType: "application/json"},
        {EntryID: "2", Event: []byte(`{"orderId": "102"}`), ContentType: "application/json"},
        {EntryID: "3", Event: []byte(`{"orderId": "103"}`), ContentType: "application/json"},
    }

    result, err := client.BulkPublishEventAlpha1(
        context.Background(),
        "order-pubsub",
        "orders",
        messages,
        map[string]string{},
    )
    if err != nil {
        panic(err)
    }

    for _, entry := range result.FailedEntries {
        // handle per-message failures
        _ = entry.EntryID
    }
}
```

## Configuring the Pub/Sub Component for Batching

Configure the underlying broker component to support batching:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-pubsub
  namespace: production
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: servicebus-secret
      key: connectionString
  - name: maxBulkSubCount
    value: "100"
  - name: maxBulkPubBytes
    value: "131072"
  - name: publishMaxRetries
    value: "3"
  - name: minConnectionRecoveryInSecs
    value: "2"
```

## Bulk Subscribe Handler

On the consumer side, register a bulk subscription handler to process batches efficiently:

```python
from dapr.ext.fastapi import DaprApp
from fastapi import FastAPI
from dapr.clients.grpc._response import TopicEventBulkResponse, TopicEventBulkResponseEntry

app = FastAPI()
dapr_app = DaprApp(app)

@dapr_app.subscribe(pubsub_name="order-pubsub", topic="orders", bulk_subscribe=True)
async def handle_bulk_orders(bulk_message):
    responses = []
    for entry in bulk_message.entries:
        order = entry.get_data()
        # process each order
        responses.append(
            TopicEventBulkResponseEntry(entry_id=entry.entry_id, status="SUCCESS")
        )
    return TopicEventBulkResponse(statuses=responses)
```

## Estimating Cost Savings

Compare API call volume before and after batching:

```bash
# Check current message throughput metrics
kubectl port-forward svc/dapr-dashboard 8080:8080 -n dapr-system
# Navigate to http://localhost:8080 and check pub/sub metrics

# Or query Prometheus directly
curl -s "http://prometheus:9090/api/v1/query?query=dapr_pubsub_publish_count[5m]"
```

For 10,000 messages per minute at $0.40 per million API calls (AWS SQS pricing):
- Without batching: 10,000 calls/min = $0.24/hr
- With batching (100 per batch): 100 calls/min = $0.0024/hr - roughly 100x savings

## Summary

Dapr's bulk publish and bulk subscribe APIs reduce message broker API calls by grouping multiple messages into single operations, directly lowering cloud messaging costs. Configure `maxBulkSubCount` on your pub/sub component and use the `BulkPublishEventAlpha1` SDK method to enable batching. For high-throughput services, batching can reduce broker costs by an order of magnitude while also improving overall publish throughput.
