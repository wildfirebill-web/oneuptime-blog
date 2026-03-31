# How to Optimize Dapr Pub/Sub Throughput

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Performance, Throughput, Optimization

Description: Learn practical techniques for maximizing Dapr Pub/Sub throughput including bulk operations, consumer concurrency, broker tuning, and message batching strategies.

---

## Understanding Pub/Sub Throughput Bottlenecks

Dapr Pub/Sub throughput is constrained by several factors: network round-trips per message, subscriber concurrency limits, broker configuration, and message serialization overhead. Identifying your bottleneck is the first step.

```bash
# Check current throughput with Dapr metrics endpoint
curl http://localhost:9090/metrics | grep dapr_component_pubsub
```

## Bulk Publishing: Reduce Network Round-Trips

Instead of publishing one message per API call, use Dapr's bulk publish API to send multiple messages at once:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"

    dapr "github.com/dapr/go-sdk/client"
)

func bulkPublish(ctx context.Context, client dapr.Client, events []OrderEvent) error {
    messages := make([]dapr.BulkPublishRequestEntry, len(events))

    for i, event := range events {
        data, _ := json.Marshal(event)
        messages[i] = dapr.BulkPublishRequestEntry{
            EntryId:     fmt.Sprintf("entry-%d", i),
            Event:       data,
            ContentType: "application/json",
        }
    }

    result, err := client.BulkPublishEventAlpha1(ctx, "messagebus", "orders", messages, nil)
    if err != nil {
        return fmt.Errorf("bulk publish failed: %w", err)
    }

    if len(result.FailedEntries) > 0 {
        fmt.Printf("Warning: %d entries failed to publish\n", len(result.FailedEntries))
    }

    fmt.Printf("Published %d messages in bulk\n", len(messages)-len(result.FailedEntries))
    return nil
}
```

## Bulk Subscribe: Process Multiple Messages Together

Enable bulk subscription to receive batches of messages in a single handler call:

```yaml
# subscriptions.yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-bulk-subscription
spec:
  pubsubname: messagebus
  topic: orders
  routes:
    default: /orders/bulk
  bulkSubscribe:
    enabled: true
    maxMessagesCount: 100
    maxAwaitDurationMs: 1000
```

Handle bulk messages in your subscriber:

```python
from fastapi import FastAPI, Request
from dapr.ext.fastapi import DaprApp
import asyncio

app = FastAPI()
dapr_app = DaprApp(app)

@app.post("/orders/bulk")
async def handle_bulk_orders(request: Request):
    body = await request.json()
    entries = body.get("entries", [])

    # Process all messages concurrently
    tasks = [process_order(entry["event"]) for entry in entries]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    statuses = []
    for entry, result in zip(entries, results):
        if isinstance(result, Exception):
            statuses.append({"entryId": entry["entryId"], "status": "RETRY"})
        else:
            statuses.append({"entryId": entry["entryId"], "status": "SUCCESS"})

    return {"statuses": statuses}
```

## Tuning Consumer Concurrency

Control how many messages your subscriber processes in parallel using a semaphore:

```python
import asyncio
from asyncio import Semaphore

# Limit to 20 concurrent message processing goroutines
semaphore = Semaphore(20)

async def process_order(event_data: dict):
    async with semaphore:
        # Actual processing here
        await do_database_write(event_data)
        await send_notification(event_data)
```

For Go services:

```go
var sem = make(chan struct{}, 20) // concurrency of 20

func processMessage(ctx context.Context, event OrderEvent) error {
    sem <- struct{}{}        // acquire
    defer func() { <-sem }() // release

    return doWork(ctx, event)
}
```

## Broker-Level Tuning

### RabbitMQ

Configure prefetch count to control how many messages the broker sends to each consumer before requiring acknowledgment:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://rabbitmq:5672"
  - name: prefetchCount
    value: "50"   # Fetch 50 messages at once (default: 0 = unlimited)
  - name: durable
    value: "true"
  - name: publisherConfirm
    value: "true"
```

### Kafka

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: messagebus
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: "kafka:9092"
  - name: consumerGroup
    value: "order-processors"
  - name: maxMessageBytes
    value: "1048576"
  - name: fetchMin
    value: "1"
  - name: fetchDefault
    value: "1048576"  # 1MB fetch batch
```

## Horizontal Scaling

Scale your subscribers horizontally to increase throughput linearly:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 5  # 5 instances for 5x throughput
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-processor"
        dapr.io/app-max-concurrency: "10"  # 10 concurrent requests per sidecar
```

Set `app-max-concurrency` to throttle the number of concurrent deliveries per pod:

```yaml
annotations:
  dapr.io/app-max-concurrency: "10"
```

## Profiling Throughput

Measure your throughput with a simple load test:

```bash
# Publish 10,000 messages and measure time
time for i in $(seq 1 10000); do
  curl -s -X POST http://localhost:3500/v1.0/publish/messagebus/orders \
    -H "Content-Type: application/json" \
    -d "{\"id\": \"order-$i\", \"total\": 9.99}" &
done
wait
```

```python
import time
import asyncio
import aiohttp

async def benchmark_throughput(num_messages: int = 10000):
    start = time.time()
    async with aiohttp.ClientSession() as session:
        tasks = [
            session.post(
                "http://localhost:3500/v1.0/publish/messagebus/orders",
                json={"id": f"order-{i}", "total": 9.99}
            )
            for i in range(num_messages)
        ]
        await asyncio.gather(*tasks)

    duration = time.time() - start
    print(f"Published {num_messages} messages in {duration:.2f}s")
    print(f"Throughput: {num_messages / duration:.0f} msg/s")

asyncio.run(benchmark_throughput())
```

## Summary

Maximizing Dapr Pub/Sub throughput requires tuning at multiple layers: use bulk publish and subscribe APIs to reduce per-message overhead, configure consumer concurrency to match your processing capacity, tune broker-specific settings like prefetch count and fetch batch size, and scale subscriber replicas horizontally. Together, these techniques can increase throughput from hundreds to tens of thousands of messages per second for typical workloads.
