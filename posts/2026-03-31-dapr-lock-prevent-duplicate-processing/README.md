# How to Use Dapr Distributed Lock for Preventing Duplicate Processing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Distributed Lock, Idempotency, Message Processing, Microservice

Description: Use Dapr distributed locks to prevent duplicate processing of events, messages, and jobs across competing microservice instances in a distributed system.

---

Duplicate message delivery is a fact of life in distributed systems. At-least-once delivery guarantees from message brokers mean your consumers may receive the same message multiple times. Dapr distributed locks provide an effective mechanism for ensuring idempotent processing without needing a dedicated deduplication store.

## The Problem

When multiple instances of a consumer service are running, they may all attempt to process the same message simultaneously. Without coordination, you get duplicate side effects - charges applied twice, emails sent multiple times, records created redundantly.

## Lock-Based Deduplication Pattern

Use the message ID as the `resourceId` to create a message-scoped lock:

```python
from dapr.clients import DaprClient
import os, socket

INSTANCE = f"{socket.gethostname()}-{os.getpid()}"

def process_message(message_id: str, payload: dict) -> bool:
    with DaprClient() as client:
        lock_resp = client.try_lock(
            store_name="lockstore",
            resource_id=f"msg-{message_id}",
            lock_owner=INSTANCE,
            expiry_in_seconds=60
        )
        if not lock_resp.success:
            print(f"Message {message_id} already being processed - skipping")
            return False

        try:
            do_process(payload)
            return True
        finally:
            client.unlock(
                store_name="lockstore",
                resource_id=f"msg-{message_id}",
                lock_owner=INSTANCE
            )
```

## Integrating with Dapr Pub/Sub

Use the lock inside your subscription handler:

```python
from dapr.ext.grpc import App

app = App()

@app.subscribe(pubsub_name="pubsub", topic="orders")
def handle_order(event) -> None:
    message_id = event.id
    payload = event.data

    success = process_message(message_id, payload)
    if success:
        print(f"Order {message_id} processed")
    else:
        print(f"Order {message_id} skipped (duplicate)")
```

## Using a Longer Expiry for Idempotency Windows

Set expiry based on how long you want to suppress duplicates:

```python
# Suppress duplicates for 5 minutes
lock_resp = client.try_lock(
    store_name="lockstore",
    resource_id=f"job-{job_id}",
    lock_owner=INSTANCE,
    expiry_in_seconds=300
)
```

After the expiry, the lock is released and a new attempt with the same ID will be processed. Choose the window based on your message broker's redelivery delay.

## Handling Transient Failures

If processing fails, you may want to release the lock so a retry can proceed:

```python
try:
    do_process(payload)
except TransientError:
    # Release lock so another instance or retry can process
    client.unlock("lockstore", f"msg-{message_id}", INSTANCE)
    raise
except PermanentError:
    # Keep lock - prevent retry processing, log for dead-letter
    log_dead_letter(message_id, payload)
```

## Metrics and Observability

Track deduplication rates to monitor health:

```python
from prometheus_client import Counter

duplicate_counter = Counter("messages_deduplicated_total", "Duplicate messages skipped")
processed_counter = Counter("messages_processed_total", "Messages processed")

def process_message(message_id, payload):
    # ... lock attempt ...
    if not lock_resp.success:
        duplicate_counter.inc()
        return False
    processed_counter.inc()
    # ... process ...
```

## Summary

Dapr distributed locks provide a simple and effective approach to preventing duplicate message processing. By using the message ID as the lock resource, each message is guaranteed to be processed at most once within the lock's expiry window. This pattern works across any number of competing consumer instances and integrates naturally with Dapr's pub/sub subscription model.
