# How to Use Redis as a Message Queue (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Message Queue, Lists, Beginner, Task Queue, Node.js, Python

Description: A beginner-friendly guide to using Redis Lists as a simple message queue, implementing producers and consumers for background task processing.

---

## What Is a Message Queue?

A message queue lets you decouple parts of your application. Instead of processing a task immediately (which might be slow), you put it in a queue, and a worker picks it up and processes it later.

```text
Your App (Producer) --> [Queue] --> Worker (Consumer)
```

Common uses: sending emails, processing images, generating reports, sending notifications.

## Redis Lists as a Queue

Redis Lists support O(1) push and pop from both ends. This makes them perfect for a simple FIFO queue:

```bash
# Producer: push task to the right end of the list
RPUSH queue:emails '{"to":"alice@example.com","subject":"Welcome!"}'

# Consumer: pop from the left end (FIFO order)
LPOP queue:emails
```

## Basic Producer in Node.js

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

async function enqueueEmail(emailData) {
  const task = {
    id: Date.now(),
    type: 'send-email',
    data: emailData,
    createdAt: new Date().toISOString()
  };

  await redis.rpush('queue:emails', JSON.stringify(task));
  console.log(`Enqueued email task: ${task.id}`);
  return task.id;
}

// Usage
await enqueueEmail({
  to: 'alice@example.com',
  subject: 'Welcome to our app!',
  body: 'Thanks for signing up.'
});

await enqueueEmail({
  to: 'bob@example.com',
  subject: 'Your order is confirmed',
  body: 'Order #12345 has been confirmed.'
});
```

## Basic Consumer in Node.js

```javascript
async function processEmailQueue() {
  console.log('Email worker started, waiting for tasks...');

  while (true) {
    // BLPOP blocks for up to 5 seconds waiting for a message
    const result = await redis.blpop('queue:emails', 5);

    if (!result) {
      console.log('No messages, still waiting...');
      continue;
    }

    const [queueName, taskJson] = result;
    const task = JSON.parse(taskJson);

    console.log(`Processing task ${task.id}: ${task.data.subject}`);

    try {
      await sendEmail(task.data); // Your email sending logic
      console.log(`Task ${task.id} completed`);
    } catch (error) {
      console.error(`Task ${task.id} failed:`, error.message);
      // Optionally re-queue or move to a dead letter queue
    }
  }
}

async function sendEmail(emailData) {
  // Simulate sending email
  console.log(`Sending email to ${emailData.to}: ${emailData.subject}`);
  await new Promise(resolve => setTimeout(resolve, 100));
}

processEmailQueue();
```

## Basic Producer in Python

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def enqueue_task(queue_name: str, task_data: dict) -> str:
    task = {
        'id': str(int(time.time() * 1000)),
        'data': task_data,
        'created_at': time.time()
    }
    r.rpush(queue_name, json.dumps(task))
    print(f"Enqueued task: {task['id']}")
    return task['id']

# Enqueue some tasks
enqueue_task('queue:emails', {'to': 'alice@example.com', 'subject': 'Welcome!'})
enqueue_task('queue:emails', {'to': 'bob@example.com', 'subject': 'Order confirmed'})
```

## Basic Consumer in Python

```python
def process_queue(queue_name: str):
    print(f"Worker started, listening on {queue_name}...")

    while True:
        # BLPOP with 5-second timeout
        result = r.blpop(queue_name, timeout=5)

        if result is None:
            print("No messages, still waiting...")
            continue

        _, task_json = result
        task = json.loads(task_json)

        print(f"Processing task {task['id']}")

        try:
            process_task(task['data'])
            print(f"Task {task['id']} done")
        except Exception as e:
            print(f"Task {task['id']} failed: {e}")

def process_task(data: dict):
    # Your processing logic here
    print(f"Sending email to {data['to']}: {data['subject']}")

process_queue('queue:emails')
```

## Checking Queue Status

```bash
# See how many tasks are waiting
LLEN queue:emails

# Peek at the next task without removing it
LINDEX queue:emails 0

# See all pending tasks
LRANGE queue:emails 0 -1
```

## Multiple Workers for Parallel Processing

You can run multiple worker processes on the same queue. BLPOP is atomic - only one worker gets each message:

```bash
# Terminal 1
node worker.js

# Terminal 2
node worker.js

# Terminal 3
node worker.js

# All three workers share the queue - each task is processed once
```

## Priority Queues with Multiple Lists

```javascript
async function enqueueWithPriority(task, priority = 'normal') {
  const queueName = `queue:emails:${priority}`; // high, normal, low
  await redis.rpush(queueName, JSON.stringify(task));
}

async function processWithPriority() {
  while (true) {
    // BLPOP checks queues in order - high priority first
    const result = await redis.blpop(
      'queue:emails:high',
      'queue:emails:normal',
      'queue:emails:low',
      5
    );

    if (result) {
      const [queue, taskJson] = result;
      const task = JSON.parse(taskJson);
      await processEmailTask(task);
    }
  }
}
```

## Monitoring Your Queue

```javascript
async function getQueueStats() {
  const [high, normal, low] = await redis.pipeline()
    .llen('queue:emails:high')
    .llen('queue:emails:normal')
    .llen('queue:emails:low')
    .exec();

  return {
    high: high[1],
    normal: normal[1],
    low: low[1],
    total: high[1] + normal[1] + low[1]
  };
}
```

## Summary

Redis Lists provide a simple, effective message queue using RPUSH to add tasks and BLPOP to receive them. Workers block on BLPOP until a message arrives, making it efficient without polling. Multiple workers can share a queue for parallel processing since BLPOP is atomic. For more advanced features like message acknowledgment and consumer groups, consider Redis Streams, but for simple background task processing, Redis Lists are perfect for beginners.
