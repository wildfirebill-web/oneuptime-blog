# How to Set Up Integration Tests for Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Testing, Pub/Sub, Integration Test, Python

Description: Write integration tests that verify Dapr pub/sub message publishing and subscription end-to-end using real brokers and Dapr sidecars in a local test environment.

---

Testing Dapr pub/sub with mocks verifies your application logic but not the subscription configuration, message delivery guarantees, or error handling for broker failures. Integration tests with real brokers catch these issues.

## Test Architecture

A pub/sub integration test environment needs:
1. A message broker (Redis Streams works well for local testing)
2. A publisher service with a Dapr sidecar
3. A subscriber service with a Dapr sidecar
4. A test harness that publishes messages and verifies delivery

## Docker Compose Setup

```yaml
# docker-compose.pubsub-test.yaml
version: "3.8"
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  publisher:
    build: ./publisher
    environment:
      - DAPR_HTTP_PORT=3500

  publisher-dapr:
    image: daprio/daprd:1.14.0
    command:
      - "./daprd"
      - "-app-id"
      - "publisher"
      - "-components-path"
      - "/components"
    volumes:
      - ./components:/components

  subscriber:
    build: ./subscriber
    ports:
      - "8081:8081"

  subscriber-dapr:
    image: daprio/daprd:1.14.0
    command:
      - "./daprd"
      - "-app-id"
      - "subscriber"
      - "-app-port"
      - "8081"
      - "-components-path"
      - "/components"
    volumes:
      - ./components:/components
```

## Pub/Sub Component for Testing

```yaml
# components/pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: test-pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: "redis:6379"
    - name: consumerID
      value: "test-consumer"
```

## Writing the Integration Test

```python
# test_pubsub.py
import requests
import time
import threading
import pytest

received_messages = []

def test_publish_and_receive():
    # Publish a message via Dapr
    resp = requests.post(
        "http://localhost:3500/v1.0/publish/test-pubsub/orders",
        json={"orderId": "integration-test-001", "amount": 99.99},
        headers={"Content-Type": "application/json"}
    )
    assert resp.status_code == 204

    # Wait for the subscriber to process the message
    deadline = time.time() + 10
    while time.time() < deadline:
        if "integration-test-001" in received_messages:
            return
        time.sleep(0.5)

    pytest.fail("Message was not received within timeout")
```

## Subscriber Endpoint

The subscriber service exposes an endpoint that Dapr calls when a message arrives:

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/dapr/subscribe", methods=["GET"])
def subscribe():
    return jsonify([{
        "pubsubname": "test-pubsub",
        "topic": "orders",
        "route": "/orders"
    }])

@app.route("/orders", methods=["POST"])
def handle_order():
    data = request.json
    received_messages.append(data.get("data", {}).get("orderId"))
    return jsonify({"status": "SUCCESS"})
```

## Testing Dead Letter Topics

Verify that failed messages are routed to the dead letter topic:

```python
def test_dead_letter_routing():
    # Publish a message that will cause processing to fail
    requests.post(
        "http://localhost:3500/v1.0/publish/test-pubsub/orders",
        json={"orderId": "FAIL_THIS"},
    )
    time.sleep(5)
    # Check the dead letter topic received the message
    assert "FAIL_THIS" in dead_letter_messages
```

## Running the Tests

```bash
docker-compose -f docker-compose.pubsub-test.yaml up -d
sleep 5  # Wait for services to be ready
pytest tests/test_pubsub.py -v
docker-compose -f docker-compose.pubsub-test.yaml down
```

## Summary

Integration tests for Dapr pub/sub verify end-to-end message delivery, subscription configuration, and error handling with real brokers. Using Redis Streams locally keeps the setup simple while providing realistic behavior that matches production Kafka or Azure Service Bus configurations.
