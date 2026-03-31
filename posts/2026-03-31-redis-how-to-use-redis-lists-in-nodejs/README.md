# How to Use Redis Lists in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Node.js, List, ioredis, Queue, Stack

Description: Learn how to use Redis Lists in Node.js with ioredis for queues, stacks, activity feeds, and blocking pops with practical real-world examples.

---

## What Are Redis Lists?

Redis Lists are ordered linked lists of strings. They support:
- O(1) push/pop from both ends
- Blocking pop operations (great for queues)
- Index-based access
- Range operations
- Trim to fixed length

Use cases: task queues, activity feeds, recent history, job queues.

## Basic List Operations

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function basicListOps() {
  // Push to the right (tail)
  await redis.rpush('tasks', 'task:1', 'task:2', 'task:3');

  // Push to the left (head)
  await redis.lpush('tasks', 'urgent:task');

  // Pop from the left (FIFO queue behavior)
  const nextTask = await redis.lpop('tasks');
  console.log(nextTask); // urgent:task

  // Pop from the right (LIFO stack behavior)
  const lastTask = await redis.rpop('tasks');
  console.log(lastTask); // task:3

  // Get all elements
  const allTasks = await redis.lrange('tasks', 0, -1);
  console.log(allTasks); // ['task:1', 'task:2']

  // Get length
  const length = await redis.llen('tasks');
  console.log(length); // 2

  // Get element by index
  const first = await redis.lindex('tasks', 0);
  console.log(first); // task:1

  // Update element at index
  await redis.lset('tasks', 0, 'modified:task:1');
}

basicListOps();
```

## Simple Task Queue

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class TaskQueue {
  constructor(name) {
    this.key = `queue:${name}`;
  }

  async enqueue(task) {
    const payload = JSON.stringify(task);
    const length = await redis.rpush(this.key, payload);
    console.log(`Task enqueued. Queue length: ${length}`);
    return length;
  }

  async dequeue() {
    const payload = await redis.lpop(this.key);
    return payload ? JSON.parse(payload) : null;
  }

  async length() {
    return redis.llen(this.key);
  }

  async peek() {
    const payload = await redis.lindex(this.key, 0);
    return payload ? JSON.parse(payload) : null;
  }

  async getAll() {
    const items = await redis.lrange(this.key, 0, -1);
    return items.map(item => JSON.parse(item));
  }
}

const queue = new TaskQueue('email');

await queue.enqueue({ to: 'alice@example.com', subject: 'Welcome' });
await queue.enqueue({ to: 'bob@example.com', subject: 'Order shipped' });

console.log('Queue length:', await queue.length());

const task = await queue.dequeue();
console.log('Processing:', task);
```

## Blocking Pop for Worker Queues

```javascript
const Redis = require('ioredis');
const worker = new Redis();

async function processWorkerQueue(queueName, processTask) {
  const queueKey = `queue:${queueName}`;
  console.log(`Worker listening on ${queueKey}...`);

  while (true) {
    try {
      // Block for up to 5 seconds waiting for a task
      const result = await worker.blpop(queueKey, 5);

      if (result) {
        const [key, payload] = result;
        const task = JSON.parse(payload);
        console.log(`Processing task from ${key}:`, task);
        await processTask(task);
      }
    } catch (err) {
      console.error('Worker error:', err);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

// Worker function
async function handleEmailTask(task) {
  console.log(`Sending email to ${task.to}: ${task.subject}`);
  // actual email sending logic here
}

// Start worker (runs indefinitely)
processWorkerQueue('email', handleEmailTask);
```

## Priority Queue with Multiple Lists

```javascript
const Redis = require('ioredis');
const redis = new Redis();
const worker = new Redis();

class PriorityQueue {
  constructor(name) {
    this.queues = {
      high: `queue:${name}:high`,
      normal: `queue:${name}:normal`,
      low: `queue:${name}:low`
    };
  }

  async enqueue(task, priority = 'normal') {
    const key = this.queues[priority] || this.queues.normal;
    await redis.rpush(key, JSON.stringify(task));
  }

  async dequeue(timeout = 0) {
    // Check queues in priority order
    const keys = [this.queues.high, this.queues.normal, this.queues.low];
    const result = await worker.blpop(...keys, timeout);

    if (result) {
      const [key, payload] = result;
      return { queue: key, task: JSON.parse(payload) };
    }
    return null;
  }
}

const pq = new PriorityQueue('jobs');
await pq.enqueue({ id: 1, type: 'email' }, 'normal');
await pq.enqueue({ id: 2, type: 'alert' }, 'high');
await pq.enqueue({ id: 3, type: 'report' }, 'low');

const next = await pq.dequeue(1);
console.log('Next job:', next); // High priority: { id: 2, type: 'alert' }
```

## Activity Feed / Recent Items

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class ActivityFeed {
  constructor(userId, maxItems = 50) {
    this.key = `feed:${userId}`;
    this.maxItems = maxItems;
  }

  async addEvent(event) {
    const payload = JSON.stringify({ ...event, ts: Date.now() });

    const pipe = redis.pipeline();
    pipe.lpush(this.key, payload);
    pipe.ltrim(this.key, 0, this.maxItems - 1); // Keep only latest N items
    await pipe.exec();
  }

  async getRecent(count = 10) {
    const items = await redis.lrange(this.key, 0, count - 1);
    return items.map(item => JSON.parse(item));
  }
}

const feed = new ActivityFeed('user:1001');

await feed.addEvent({ type: 'login', device: 'web' });
await feed.addEvent({ type: 'purchase', itemId: 'prod:42', amount: 29.99 });
await feed.addEvent({ type: 'review', itemId: 'prod:42', rating: 5 });

const recent = await feed.getRecent(5);
console.log('Recent activity:');
recent.forEach(e => console.log(`  [${e.type}]`, e.ts));
```

## Trimming Lists to Fixed Size

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function addToFixedList(key, value, maxSize = 100) {
  const pipe = redis.pipeline();
  pipe.rpush(key, value);
  pipe.ltrim(key, -maxSize, -1); // Keep last N items
  await pipe.exec();
}

// Keep rolling window of last 1000 log entries
for (let i = 0; i < 1500; i++) {
  await addToFixedList('logs:recent', `log entry ${i}`, 1000);
}

const length = await redis.llen('logs:recent');
console.log('Log entries retained:', length); // 1000
```

## Summary

Redis Lists in Node.js with ioredis support both stack (LIFO) and queue (FIFO) patterns via `lpush`/`rpush` and `lpop`/`rpop`. Blocking pop with `blpop` enables efficient worker queue patterns without polling. Combine `lpush` with `ltrim` for fixed-size activity feeds, and use multiple priority queues with a single `blpop` call across all queues for simple priority scheduling.
