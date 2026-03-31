# How to Implement Priority Queue with Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Priority Queue, Messaging, Distributed Systems

Description: Learn how to implement a priority queue pattern using Dapr Pub/Sub with multiple topics mapped to different processing priorities and consumer configurations.

---

## Why Priority Queues Matter in Messaging

A standard pub/sub system treats all messages equally. A priority queue ensures that critical messages - like payment alerts or security events - are processed before lower-priority ones like analytics updates. Dapr Pub/Sub supports priority queue patterns through multiple topic routing and consumer configuration.

## Architecture Overview

The priority queue pattern with Dapr uses separate topics for each priority level:

```text
Publishers                    Dapr Pub/Sub          Consumers

[Payment Service] -------> [topic: payments-high] --> [High Priority Worker]
[Order Service]   -------> [topic: orders-medium] --> [Medium Priority Worker]
[Analytics]       -------> [topic: events-low]    --> [Low Priority Worker]
```

## Configuring Pub/Sub Component

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
  - name: prefetchCount
    value: "10"
```

## Publishing Messages with Priority

Publish to different topics based on priority level:

```python
import json
import dapr.clients as dapr_client
from enum import Enum

class Priority(Enum):
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

PRIORITY_TOPICS = {
    Priority.HIGH:   "messages-high",
    Priority.MEDIUM: "messages-medium",
    Priority.LOW:    "messages-low",
}

def publish_message(message: dict, priority: Priority):
    with dapr_client.DaprClient() as client:
        topic = PRIORITY_TOPICS[priority]
        client.publish_event(
            pubsub_name="messagebus",
            topic_name=topic,
            data=json.dumps(message),
            data_content_type="application/json",
            publish_metadata={"ttlInSeconds": "3600"}
        )
        print(f"Published to {topic}: {message}")

# Usage
publish_message(
    {"type": "payment_failed", "order_id": "ORD-1001", "amount": 99.99},
    Priority.HIGH
)

publish_message(
    {"type": "order_shipped", "order_id": "ORD-1002"},
    Priority.MEDIUM
)

publish_message(
    {"type": "page_view", "user_id": "user-42", "page": "/home"},
    Priority.LOW
)
```

## Subscribing to Priority Topics

Each priority level can have a dedicated consumer with different concurrency settings:

```python
from fastapi import FastAPI
from dapr.ext.fastapi import DaprApp

app = FastAPI()
dapr_app = DaprApp(app)

# High priority - process immediately, limited backlog
@dapr_app.subscribe(pubsub="messagebus", topic="messages-high")
async def handle_high_priority(event_data: dict):
    print(f"[HIGH PRIORITY] Processing: {event_data}")
    await process_high_priority_event(event_data)
    return {"success": True}

# Medium priority - normal processing
@dapr_app.subscribe(pubsub="messagebus", topic="messages-medium")
async def handle_medium_priority(event_data: dict):
    print(f"[MEDIUM] Processing: {event_data}")
    await process_medium_priority_event(event_data)
    return {"success": True}

# Low priority - can be delayed, batch-processed
@dapr_app.subscribe(pubsub="messagebus", topic="messages-low")
async def handle_low_priority(event_data: dict):
    print(f"[LOW] Processing: {event_data}")
    await process_low_priority_event(event_data)
    return {"success": True}
```

## Programmatic Subscription with Bulk Settings

For fine-grained control, use programmatic subscription to configure bulk settings per priority:

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/dapr/go-sdk/service/common"
    daprd "github.com/dapr/go-sdk/service/http"
)

func main() {
    s := daprd.NewService(":6002")

    // High priority: low max items, short wait time - process quickly
    s.AddBulkTopicEventHandler(&common.Subscription{
        PubsubName: "messagebus",
        Topic:      "messages-high",
        BulkSubscribeConfig: common.BulkSubscribeConfig{
            MaxMessagesCount:   5,
            MaxAwaitDurationMs: 100,
        },
    }, handleHighPriority)

    // Low priority: high max items, longer wait - batch for efficiency
    s.AddBulkTopicEventHandler(&common.Subscription{
        PubsubName: "messagebus",
        Topic:      "messages-low",
        BulkSubscribeConfig: common.BulkSubscribeConfig{
            MaxMessagesCount:   100,
            MaxAwaitDurationMs: 5000,
        },
    }, handleLowPriority)

    log.Fatal(s.Start())
}
```

## Priority Router: Single Consumer, Multiple Priorities

An alternative approach uses a single consumer that routes messages by priority internally:

```python
import asyncio
from collections import deque

class PriorityMessageRouter:
    def __init__(self):
        self.queues = {
            "high":   deque(),
            "medium": deque(),
            "low":    deque(),
        }

    def enqueue(self, message: dict, priority: str):
        self.queues[priority].appendleft(message)

    async def process_loop(self):
        while True:
            # Always drain high priority first
            if self.queues["high"]:
                msg = self.queues["high"].pop()
                await self.process(msg, "high")
            elif self.queues["medium"]:
                msg = self.queues["medium"].pop()
                await self.process(msg, "medium")
            elif self.queues["low"]:
                msg = self.queues["low"].pop()
                await self.process(msg, "low")
            else:
                await asyncio.sleep(0.1)

    async def process(self, message: dict, priority: str):
        print(f"[{priority.upper()}] Processing: {message}")
        # Add actual processing logic here
```

## Monitoring Priority Queue Depths

Monitor the depth of each priority topic using Dapr's metrics:

```bash
# Check RabbitMQ queue depths (adjust for your broker)
curl -s http://localhost:15672/api/queues | \
  python3 -c "
import sys, json
queues = json.load(sys.stdin)
for q in queues:
    if 'messages' in q['name']:
        print(f\"{q['name']}: {q['messages']} pending\")
"
```

## Summary

Dapr Pub/Sub supports a priority queue pattern by using separate topics for each priority level and configuring consumers with different bulk settings and concurrency. High-priority topics get dedicated consumers with low latency settings, while low-priority topics use batching for efficiency. This approach cleanly separates critical from non-critical message flows without requiring a specialized priority queue broker.
