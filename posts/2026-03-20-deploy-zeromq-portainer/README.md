# How to Deploy ZeroMQ-Based Applications via Portainer - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, ZeroMQ, Messaging, Docker, Microservice

Description: Deploy applications using ZeroMQ for high-performance asynchronous messaging via Portainer, covering PUB/SUB, PUSH/PULL, and REQ/REP patterns.

## Introduction

ZeroMQ (ZMQ) is a lightweight, high-performance messaging library that runs inside application processes - it has no separate broker process. It provides building blocks for message patterns: PUB/SUB (broadcast), PUSH/PULL (pipeline), and REQ/REP (request-reply). This guide deploys ZeroMQ-based microservices via Portainer.

## ZeroMQ Patterns Overview

| Pattern | Use Case | Socket Types |
|---------|----------|-------------|
| PUB/SUB | Event broadcasting | `zmq.PUB` / `zmq.SUB` |
| PUSH/PULL | Work distribution (pipeline) | `zmq.PUSH` / `zmq.PULL` |
| REQ/REP | Remote procedure call | `zmq.REQ` / `zmq.REP` |

## Step 1: Build Application Images with ZeroMQ

Create a `Dockerfile` for your ZeroMQ service:

```dockerfile
# Dockerfile - Python service with ZeroMQ

FROM python:3.12-slim

WORKDIR /app
RUN pip install pyzmq

COPY publisher.py .
CMD ["python", "publisher.py"]
```

```python
# publisher.py - ZeroMQ PUB/SUB publisher
import zmq
import json
import time

context = zmq.Context()
socket = context.socket(zmq.PUB)
socket.bind("tcp://0.0.0.0:5556")

print("Publisher started on port 5556")

while True:
    event = {
        "type": "order.created",
        "order_id": 1234,
        "timestamp": time.time()
    }
    # Topic-based routing: "orders " prefix
    socket.send_string(f"orders {json.dumps(event)}")
    time.sleep(1)
```

```python
# subscriber.py - ZeroMQ PUB/SUB subscriber
import zmq
import json

context = zmq.Context()
socket = context.socket(zmq.SUB)
socket.connect("tcp://publisher:5556")
socket.setsockopt_string(zmq.SUBSCRIBE, "orders")  # Subscribe to "orders" topic

print("Subscriber connected, waiting for messages...")

while True:
    message = socket.recv_string()
    topic, data = message.split(" ", 1)
    event = json.loads(data)
    print(f"Received {topic}: {event}")
```

## Step 2: Create the Stack in Portainer

```yaml
# docker-compose.yml - ZeroMQ PUB/SUB Architecture
version: "3.8"

services:
  publisher:
    build:
      context: ./publisher
      dockerfile: Dockerfile
    container_name: zmq_publisher
    restart: unless-stopped
    expose:
      - "5556"    # PUB socket (do not expose externally)
    networks:
      - zmq_net

  subscriber_1:
    build:
      context: ./subscriber
      dockerfile: Dockerfile
    container_name: zmq_subscriber_1
    restart: unless-stopped
    environment:
      - SUBSCRIBER_ID=1
    networks:
      - zmq_net
    depends_on:
      - publisher

  subscriber_2:
    build:
      context: ./subscriber
      dockerfile: Dockerfile
    container_name: zmq_subscriber_2
    restart: unless-stopped
    environment:
      - SUBSCRIBER_ID=2
    networks:
      - zmq_net
    depends_on:
      - publisher

networks:
  zmq_net:
    driver: bridge
```

## Step 3: PUSH/PULL Worker Pool

For task distribution across worker containers:

```python
# task_producer.py - PUSH tasks to workers
import zmq
import json

context = zmq.Context()
socket = context.socket(zmq.PUSH)
socket.bind("tcp://0.0.0.0:5557")

for i in range(100):
    task = {"id": i, "data": f"process item {i}"}
    socket.send_json(task)
    print(f"Sent task {i}")
```

```python
# worker.py - PULL tasks and process them
import zmq
import os

context = zmq.Context()
socket = context.socket(zmq.PULL)
socket.connect("tcp://task_producer:5557")

worker_id = os.getenv("WORKER_ID", "1")
print(f"Worker {worker_id} ready")

while True:
    task = socket.recv_json()
    print(f"Worker {worker_id} processing task {task['id']}")
    # Process task here
```

## Step 4: Test Connectivity

```bash
# Check publisher is sending
docker logs zmq_publisher --tail 10

# Check subscriber received messages
docker logs zmq_subscriber_1 --tail 10

# Test ZeroMQ port from inside the network
docker exec zmq_subscriber_1 python3 -c "
import zmq
ctx = zmq.Context()
s = ctx.socket(zmq.SUB)
s.connect('tcp://publisher:5556')
s.setsockopt_string(zmq.SUBSCRIBE, '')
msg = s.recv_string(flags=zmq.NOBLOCK)
print('Received:', msg)
"
```

## Conclusion

ZeroMQ runs inside application processes with no separate broker, making it extremely fast (millions of messages/second) and operationally simple. Use PUB/SUB for broadcasting events to multiple consumers, PUSH/PULL for distributing work across a pool of workers, and REQ/REP for synchronous request-reply RPCs. In Docker environments, use Docker service names as ZeroMQ endpoints - subscribers `connect()` to publishers and workers `connect()` to task producers. Never expose ZeroMQ ports externally; keep them within Docker networks.
