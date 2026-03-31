# How to Use MongoDB with RabbitMQ for Task Queues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, RabbitMQ, Queue, Task, Messaging

Description: Learn how to use MongoDB and RabbitMQ together to build reliable task queue systems, storing task state in MongoDB and routing work via RabbitMQ.

---

## Overview

MongoDB and RabbitMQ are a natural pairing for task queue systems. RabbitMQ handles message routing, delivery guarantees, and worker coordination, while MongoDB stores task state, results, and history. This architecture separates concerns cleanly: RabbitMQ is ephemeral and fast; MongoDB is durable and queryable.

## Architecture Overview

```text
Architecture:
Producer -> RabbitMQ Exchange -> Queue -> Worker(s) -> MongoDB (task state)

Flow:
1. Producer enqueues a task message in RabbitMQ (task ID + payload)
2. Worker receives the message, marks task "in_progress" in MongoDB
3. Worker processes the task
4. Worker marks task "completed" or "failed" in MongoDB
5. Worker acknowledges or rejects the RabbitMQ message
```

## Setup RabbitMQ and MongoDB

```bash
# Start RabbitMQ and MongoDB with Docker Compose
cat > docker-compose.yml << 'EOF'
version: "3.8"
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
  mongodb:
    image: mongo:7
    ports: ["27017:27017"]
EOF

docker-compose up -d
```

## MongoDB Task Schema

```javascript
// MongoDB tasks collection schema
db.tasks.createIndex({ status: 1, createdAt: 1 });
db.tasks.createIndex({ taskId: 1 }, { unique: true });

// Task document shape
{
  _id: ObjectId(),
  taskId: "uuid-here",
  type: "send_email",
  payload: { to: "user@example.com", subject: "Welcome" },
  status: "pending",   // pending | in_progress | completed | failed
  attempts: 0,
  result: null,
  error: null,
  createdAt: new Date(),
  updatedAt: new Date()
}
```

## Producer: Enqueue Tasks

```python
import pika, json, uuid
from pymongo import MongoClient
from datetime import datetime, timezone

mongo = MongoClient("mongodb://localhost:27017")
db = mongo["taskqueue"]

connection = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
channel = connection.channel()
channel.queue_declare(queue="tasks", durable=True)

def enqueue_task(task_type, payload):
  task_id = str(uuid.uuid4())
  # Store task in MongoDB
  db.tasks.insert_one({
    "taskId": task_id,
    "type": task_type,
    "payload": payload,
    "status": "pending",
    "attempts": 0,
    "createdAt": datetime.now(timezone.utc)
  })
  # Publish task ID to RabbitMQ
  channel.basic_publish(
    exchange="",
    routing_key="tasks",
    body=json.dumps({"taskId": task_id}),
    properties=pika.BasicProperties(delivery_mode=2)  # persistent
  )
  print(f"Enqueued task: {task_id}")
  return task_id

enqueue_task("send_email", {"to": "alice@example.com", "subject": "Welcome"})
```

## Worker: Process Tasks

```python
import pika, json
from pymongo import MongoClient
from datetime import datetime, timezone

mongo = MongoClient("mongodb://localhost:27017")
db = mongo["taskqueue"]

def process_task(task):
  if task["type"] == "send_email":
    # Simulate sending email
    print(f"Sending email to {task['payload']['to']}")
    return {"sent": True}
  raise ValueError(f"Unknown task type: {task['type']}")

def on_message(channel, method, properties, body):
  msg = json.loads(body)
  task_id = msg["taskId"]

  # Mark task in_progress in MongoDB
  db.tasks.update_one(
    {"taskId": task_id},
    {"$set": {"status": "in_progress", "updatedAt": datetime.now(timezone.utc)},
     "$inc": {"attempts": 1}}
  )

  try:
    task = db.tasks.find_one({"taskId": task_id})
    result = process_task(task)
    db.tasks.update_one(
      {"taskId": task_id},
      {"$set": {"status": "completed", "result": result, "updatedAt": datetime.now(timezone.utc)}}
    )
    channel.basic_ack(method.delivery_tag)
  except Exception as e:
    db.tasks.update_one(
      {"taskId": task_id},
      {"$set": {"status": "failed", "error": str(e), "updatedAt": datetime.now(timezone.utc)}}
    )
    channel.basic_nack(method.delivery_tag, requeue=False)

connection = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
channel = connection.channel()
channel.queue_declare(queue="tasks", durable=True)
channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue="tasks", on_message_callback=on_message)
print("Worker started")
channel.start_consuming()
```

## Query Task Status from MongoDB

```javascript
// Query task status
db.tasks.find({ status: "failed" }).sort({ createdAt: -1 }).limit(10);

// Aggregate task stats
db.tasks.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } }
]);
```

## Summary

Combining MongoDB with RabbitMQ creates a robust task queue system where RabbitMQ handles routing and worker coordination while MongoDB provides durable, queryable task state. Store task metadata in MongoDB before publishing to RabbitMQ, update status atomically in the worker, and use `basic_ack`/`basic_nack` for reliable delivery. MongoDB's query capabilities make it easy to monitor, retry, and audit task history.
