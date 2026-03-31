# How to Implement Request Deduplication Across Services with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Request Deduplication, Idempotency, Microservices, Distributed Systems

Description: Implement request deduplication using Redis to prevent duplicate processing of API calls, webhooks, and messages in distributed microservice architectures.

---

## Overview

In distributed systems, requests may be retried due to network failures, timeouts, or client bugs. Without deduplication, duplicate requests can cause double charges, duplicate orders, or corrupted state. Redis provides idempotency keys with TTLs to detect and suppress duplicate requests efficiently.

## Idempotency Key Pattern

```bash
# Store request result with a deduplication key
SET idempotency:{key} {result_json} EX 86400

# Check if already processed
EXISTS idempotency:{key}

# Atomic check-and-set with NX (only set if not exists)
SET idempotency:{key} "processing" NX EX 86400
```

## Core Deduplication Logic

```python
import json
import time
import hashlib
from redis import Redis
from enum import Enum

r = Redis(host='localhost', port=6379, decode_responses=True)

DEDUP_TTL = 86400  # 24 hours
PROCESSING_TTL = 60  # 60 seconds lock during processing

class RequestStatus(str, Enum):
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

def generate_idempotency_key(
    service: str,
    operation: str,
    params: dict
) -> str:
    """Generate a consistent idempotency key from request params."""
    canonical = json.dumps(params, sort_keys=True)
    digest = hashlib.sha256(canonical.encode()).hexdigest()[:16]
    return f"idempotency:{service}:{operation}:{digest}"

def process_with_dedup(
    service: str,
    operation: str,
    params: dict,
    handler_fn
) -> dict:
    """Execute handler only once per unique request."""
    key = generate_idempotency_key(service, operation, params)

    # Atomically acquire processing lock
    acquired = r.set(key, RequestStatus.PROCESSING, nx=True, ex=PROCESSING_TTL)

    if not acquired:
        # Key already exists - check status
        stored = r.get(key)

        if stored == RequestStatus.PROCESSING:
            # Another instance is processing - return pending status
            return {"status": "pending", "idempotency_key": key}

        try:
            # Return cached result
            return json.loads(stored)
        except (json.JSONDecodeError, TypeError):
            pass

    try:
        # Execute the actual operation
        result = handler_fn(params)
        response = {
            "status": RequestStatus.COMPLETED,
            "result": result,
            "processed_at": int(time.time())
        }
        # Store result with full TTL
        r.set(key, json.dumps(response), ex=DEDUP_TTL)
        return response
    except Exception as e:
        error_response = {
            "status": RequestStatus.FAILED,
            "error": str(e),
            "failed_at": int(time.time())
        }
        r.set(key, json.dumps(error_response), ex=DEDUP_TTL)
        raise
```

## Payment Deduplication Example

```python
def process_payment(payment_data: dict) -> dict:
    """Process a payment with deduplication."""
    def do_charge(params: dict) -> dict:
        # Actual payment processing logic
        charge_id = f"charge_{int(time.time())}"
        return {
            "charge_id": charge_id,
            "amount": params["amount"],
            "currency": params["currency"],
            "status": "succeeded"
        }

    return process_with_dedup(
        service="payments",
        operation="charge",
        params={
            "user_id": payment_data["user_id"],
            "amount": payment_data["amount"],
            "currency": payment_data["currency"],
            "order_id": payment_data["order_id"]
        },
        handler_fn=do_charge
    )

# First call - processes payment
result1 = process_payment({
    "user_id": "user_123",
    "amount": 5000,
    "currency": "usd",
    "order_id": "order_abc"
})

# Retry with same data - returns cached result, no double charge
result2 = process_payment({
    "user_id": "user_123",
    "amount": 5000,
    "currency": "usd",
    "order_id": "order_abc"
})
```

## Webhook Deduplication

```python
def deduplicate_webhook(
    webhook_id: str,
    payload: dict,
    handler
) -> bool:
    """Deduplicate webhook delivery using webhook ID."""
    key = f"webhook:{webhook_id}"

    # Use NX to atomically claim this webhook
    acquired = r.set(key, "processing", nx=True, ex=DEDUP_TTL)
    if not acquired:
        return False  # Already processed

    try:
        handler(payload)
        r.set(key, "processed", ex=DEDUP_TTL)
        return True
    except Exception:
        r.delete(key)  # Allow retry on failure
        raise

# Usage with Stripe webhook
def handle_stripe_event(event_id: str, event_payload: dict):
    def process(payload):
        event_type = payload.get("type")
        if event_type == "payment_intent.succeeded":
            fulfill_order(payload["data"]["object"])

    deduplicate_webhook(event_id, event_payload, process)
```

## Message Queue Deduplication

```python
def consume_message_once(
    message_id: str,
    message_data: dict,
    processor
) -> bool:
    """Process a queue message exactly once."""
    dedup_key = f"msg:processed:{message_id}"

    # NX ensures only one consumer processes this message
    acquired = r.set(dedup_key, "1", nx=True, ex=3600)
    if not acquired:
        return False  # Already processed by another consumer

    processor(message_data)
    return True
```

## Client-Side Idempotency Key

```python
import uuid

def make_idempotent_request(
    api_client,
    endpoint: str,
    data: dict,
    idempotency_key: str = None
) -> dict:
    """Make an API request with an idempotency key."""
    if not idempotency_key:
        idempotency_key = str(uuid.uuid4())

    response = api_client.post(
        endpoint,
        json=data,
        headers={"Idempotency-Key": idempotency_key}
    )
    return response.json()

# FastAPI server-side handling
from fastapi import FastAPI, Request, Header

app = FastAPI()

@app.post("/payments/charge")
async def charge_endpoint(
    request: Request,
    idempotency_key: str = Header(None, alias="Idempotency-Key")
):
    payment_data = await request.json()
    if idempotency_key:
        cache_key = f"idempotency:{idempotency_key}"
        cached = r.get(cache_key)
        if cached:
            return json.loads(cached)

    result = process_payment(payment_data)

    if idempotency_key:
        r.set(cache_key, json.dumps(result), ex=DEDUP_TTL)

    return result
```

## Summary

Redis idempotency keys prevent duplicate request processing by storing operation results keyed by a unique request identifier with an appropriate TTL. The atomic `SET NX` operation ensures only one process executes an operation even with concurrent retries. This pattern is essential for payment processing, webhook handling, and any distributed operation where duplicates would cause data corruption or incorrect charges.
