# How to Implement Message Deduplication with Dapr State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Message Deduplication, Idempotency, Pub/Sub

Description: Learn how to use Dapr State Store to implement idempotent message processing, preventing duplicate events from being processed more than once.

---

## The Duplicate Message Problem

In distributed systems, message brokers guarantee at-least-once delivery. This means your consumers may receive the same message more than once due to network retries, consumer crashes, or broker redelivery. Without deduplication, you risk double-charging customers, creating duplicate records, or triggering the same action multiple times.

Dapr State Store provides an ideal deduplication backend because it offers atomic operations and can be shared across service instances.

## Deduplication Strategy

The most common approach: store a unique message ID in state with a TTL. Before processing, check if the ID exists. If it does, skip. If it does not, store it and process.

```text
Receive message with ID "msg-abc123"
    |
    v
Check state store: "dedup:msg-abc123" exists?
    |               |
    |               YES -> Skip, return success
    NO
    |
    v
Process the message
    |
    v
Save "dedup:msg-abc123" to state store (TTL: 24h)
```

## Implementing the Deduplication Check

```python
import json
from dapr.clients import DaprClient

def process_with_dedup(message: dict, message_id: str) -> dict:
    """Process a message exactly once using Dapr state store for deduplication."""
    dedup_key = f"dedup:{message_id}"
    store_name = "statestore"

    with DaprClient() as client:
        # Check if we've already processed this message
        existing = client.get_state(store_name, dedup_key)
        if existing.data:
            print(f"Duplicate message {message_id} - skipping")
            return {"status": "duplicate", "message_id": message_id}

        # Process the message
        result = handle_message(message)

        # Mark as processed with a 24-hour TTL
        client.save_state(
            store_name=store_name,
            key=dedup_key,
            value=json.dumps({
                "processed_at": datetime.utcnow().isoformat(),
                "result": result
            }),
            state_metadata={"ttlInSeconds": "86400"}  # 24 hours
        )

        print(f"Processed message {message_id}")
        return {"status": "processed", "result": result}

def handle_message(message: dict) -> dict:
    """Actual business logic for processing the message."""
    event_type = message.get("event_type")
    if event_type == "payment":
        return process_payment(message)
    elif event_type == "order":
        return process_order(message)
    return {"handled": True}
```

## Integrating with Dapr Pub/Sub

Use the deduplication wrapper in your pub/sub subscriber:

```python
from fastapi import FastAPI
from dapr.ext.fastapi import DaprApp
import json

app = FastAPI()
dapr_app = DaprApp(app)

@dapr_app.subscribe(pubsub="messagebus", topic="payments")
async def handle_payment_event(event_data: dict):
    # Dapr provides the message ID in the CloudEvent envelope
    message_id = event_data.get("id")
    if not message_id:
        print("Warning: No message ID found, cannot deduplicate")
        message_id = f"unknown-{hash(json.dumps(event_data))}"

    result = process_with_dedup(event_data.get("data", {}), message_id)
    return result
```

## Atomic Dedup Using First-Write-Wins

For stricter guarantees, use Dapr's first-write-wins concurrency to atomically claim a message:

```go
package main

import (
    "context"
    "fmt"
    "strings"

    dapr "github.com/dapr/go-sdk/client"
)

func processOnce(ctx context.Context, client dapr.Client, messageID string, payload []byte) error {
    dedupKey := "dedup:" + messageID
    storeName := "statestore"

    // Try to claim the message atomically with first-write-wins
    err := client.SaveState(ctx, storeName, dedupKey, payload, map[string]string{
        "concurrency":  "first-write",
        "ttlInSeconds": "86400",
    })

    if err != nil {
        if strings.Contains(err.Error(), "conflict") ||
            strings.Contains(err.Error(), "etag mismatch") {
            fmt.Printf("Message %s already claimed by another instance\n", messageID)
            return nil // Not an error - just a duplicate
        }
        return fmt.Errorf("failed to claim message: %w", err)
    }

    // We won the race - process the message
    return doActualWork(ctx, payload)
}
```

## Handling the TTL Expiry Edge Case

When you use TTLs for dedup records, there is a small window: if a message arrives after the TTL expires, it will be re-processed. Set your TTL based on how long your message broker retains unacknowledged messages:

```python
def calculate_dedup_ttl(broker_retention_seconds: int) -> int:
    """
    TTL should be longer than the broker's retry window.
    Add a buffer to handle clock skew between services.
    """
    buffer_seconds = 3600  # 1 hour buffer
    return broker_retention_seconds + buffer_seconds

# For a broker that retries for 24 hours, use a 25-hour TTL
TTL = calculate_dedup_ttl(86400)
```

## Batch Deduplication Check

For high-throughput scenarios, use bulk state operations to check multiple message IDs at once:

```python
def batch_dedup_check(message_ids: list, client: DaprClient) -> set:
    """Returns set of message IDs that have already been processed."""
    keys = [f"dedup:{mid}" for mid in message_ids]
    items = client.get_bulk_state("statestore", keys, parallelism=10)

    already_processed = set()
    for item in items:
        if item.data:
            original_id = item.key.replace("dedup:", "")
            already_processed.add(original_id)

    return already_processed

def process_batch(messages: list):
    with DaprClient() as client:
        ids = [m["id"] for m in messages]
        duplicates = batch_dedup_check(ids, client)

        for message in messages:
            if message["id"] in duplicates:
                print(f"Skipping duplicate {message['id']}")
                continue
            process_with_dedup(message, message["id"])
```

## Summary

Dapr State Store provides a simple and effective backend for implementing message deduplication in distributed systems. By storing message IDs with a TTL, you can efficiently detect and skip duplicate events. For concurrent scenarios where multiple service instances compete, first-write-wins concurrency ensures exactly one instance processes each message. Combined with Dapr Pub/Sub, this pattern gives you reliable at-least-once delivery semantics with exactly-once processing behavior.
