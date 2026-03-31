# How to Use Redis Lists for Task Queues (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lists, Task Queue, Beginner, BLPOP, Node.js, Python

Description: A beginner-friendly guide to building task queues with Redis Lists, covering RPUSH, BLPOP, list inspection, and basic worker patterns.

---

## What Are Redis Lists?

A Redis List is an ordered sequence of strings. You can push items to either end and pop from either end. For task queues, you push new tasks to one end and workers pop from the other.

```bash
# Add tasks to the right end (RPUSH = queue-style, FIFO)
RPUSH tasks "send-email"
RPUSH tasks "resize-image"
RPUSH tasks "generate-report"

# Pop from the left end (oldest task first)
LPOP tasks
# "send-email"
```

## Basic List Commands

```bash
# Push one or multiple values to the right
RPUSH mylist "first" "second" "third"

# Push to the left (front of list)
LPUSH mylist "prepended"

# Pop from the left (removes and returns)
LPOP mylist

# Pop from the right
RPOP mylist

# View without removing
LINDEX mylist 0   # First element
LINDEX mylist -1  # Last element

# Get a range (0 to 4 = first 5 elements)
LRANGE mylist 0 4

# Count items
LLEN mylist
```

## FIFO Queue (First In, First Out)

Tasks are processed in the order they were added:

```bash
RPUSH queue "task-1"   # Add to tail
RPUSH queue "task-2"
RPUSH queue "task-3"

LPOP queue  # "task-1" (oldest first)
LPOP queue  # "task-2"
LPOP queue  # "task-3"
```

## LIFO Stack (Last In, First Out)

```bash
RPUSH stack "task-1"
RPUSH stack "task-2"

RPOP stack  # "task-2" (newest first)
RPOP stack  # "task-1"
```

## BLPOP - Blocking Pop (The Worker Pattern)

`BLPOP` waits until a task is available instead of returning nil immediately:

```bash
# Block for up to 10 seconds waiting for a task
BLPOP queue:tasks 10

# Block indefinitely (0 = no timeout)
BLPOP queue:tasks 0
```

## Node.js Task Queue Worker

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

// Producer: Add tasks to the queue
async function addTask(queueName, taskData) {
  const task = {
    id: `task-${Date.now()}`,
    createdAt: new Date().toISOString(),
    data: taskData
  };
  await redis.rpush(queueName, JSON.stringify(task));
  console.log(`Added task: ${task.id}`);
  return task.id;
}

// Consumer: Process tasks continuously
async function runWorker(queueName) {
  console.log(`Worker started on queue: ${queueName}`);

  while (true) {
    // Wait up to 5 seconds for a new task
    const result = await redis.blpop(queueName, 5);

    if (!result) continue; // Timeout, loop again

    const [queue, taskJson] = result;
    const task = JSON.parse(taskJson);

    console.log(`Processing: ${task.id}`);
    await handleTask(task);
  }
}

async function handleTask(task) {
  const { type, payload } = task.data;

  switch (type) {
    case 'send-email':
      console.log(`Sending email to ${payload.to}`);
      break;
    case 'resize-image':
      console.log(`Resizing image: ${payload.imageUrl}`);
      break;
    default:
      console.log(`Unknown task type: ${type}`);
  }
}

// Enqueue some tasks
await addTask('queue:tasks', { type: 'send-email', payload: { to: 'alice@example.com' } });
await addTask('queue:tasks', { type: 'resize-image', payload: { imageUrl: '/img/photo.jpg' } });

// Start worker
runWorker('queue:tasks');
```

## Python Task Queue

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def enqueue(queue_name: str, task_type: str, payload: dict) -> str:
    task = {
        'id': f"task-{int(time.time() * 1000)}",
        'type': task_type,
        'payload': payload,
        'created_at': time.time()
    }
    r.rpush(queue_name, json.dumps(task))
    print(f"Enqueued: {task['id']} ({task_type})")
    return task['id']

def run_worker(queue_name: str):
    print(f"Worker listening on {queue_name}...")

    while True:
        result = r.blpop(queue_name, timeout=5)

        if result is None:
            continue

        _, task_json = result
        task = json.loads(task_json)

        print(f"Processing: {task['id']} ({task['type']})")
        process_task(task)

def process_task(task: dict):
    task_type = task['type']
    payload = task['payload']

    if task_type == 'send-email':
        print(f"Sending email to: {payload['to']}")
    elif task_type == 'generate-report':
        print(f"Generating report: {payload['report_id']}")
    else:
        print(f"Unknown task type: {task_type}")

# Enqueue tasks
enqueue('queue:tasks', 'send-email', {'to': 'alice@example.com'})
enqueue('queue:tasks', 'generate-report', {'report_id': 'rpt-456'})

# Start processing
run_worker('queue:tasks')
```

## Inspecting the Queue

```bash
# How many tasks are waiting?
LLEN queue:tasks

# Preview next task without removing it
LINDEX queue:tasks 0

# View all pending tasks (careful on large queues)
LRANGE queue:tasks 0 -1

# View first 5 pending tasks
LRANGE queue:tasks 0 4
```

## Multiple Queues with Priority

```javascript
async function runPriorityWorker() {
  while (true) {
    // BLPOP checks queues in order - high priority first
    const result = await redis.blpop(
      'queue:high',
      'queue:normal',
      'queue:low',
      5
    );

    if (result) {
      const [queue, taskJson] = result;
      console.log(`Processing from ${queue}`);
      await handleTask(JSON.parse(taskJson));
    }
  }
}

// Enqueue to different priority levels
await redis.rpush('queue:high', JSON.stringify({ type: 'alert', data: 'critical-error' }));
await redis.rpush('queue:normal', JSON.stringify({ type: 'email', data: 'welcome' }));
await redis.rpush('queue:low', JSON.stringify({ type: 'report', data: 'weekly' }));
```

## Summary

Redis Lists are a simple, effective foundation for task queues. RPUSH adds tasks to the tail and BLPOP blocks workers until tasks are available - eliminating polling. Workers can run in parallel; BLPOP is atomic so each task is claimed by exactly one worker. For priority queues, pass multiple list names to BLPOP and Redis checks them in order. For basic background processing without needing acknowledgment or consumer groups, Redis Lists are the simplest starting point.
