# How to Use BLPOP and BRPOP in Redis for Blocking List Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lists, BLPOP, BRPOP, Queues

Description: Learn how to use BLPOP and BRPOP in Redis to block and wait for list elements, enabling efficient event-driven queue consumers without polling.

---

## What Are BLPOP and BRPOP

`BLPOP` and `BRPOP` are the blocking variants of `LPOP` and `RPOP`. When called on an empty list, instead of returning nil immediately, they block the connection and wait until an element is pushed or a timeout expires.

- `BLPOP` - blocks and pops from the left (head) of the first non-empty list
- `BRPOP` - blocks and pops from the right (tail) of the first non-empty list

Both accept multiple keys and check them left to right, returning from the first key that becomes non-empty.

## Syntax

```text
BLPOP key [key ...] timeout
BRPOP key [key ...] timeout
```

- `key [key ...]` - one or more list keys to monitor
- `timeout` - seconds to block; `0` blocks indefinitely; decimals like `1.5` are supported since Redis 6.0

Returns a two-element array `[key, value]` - the key from which the element was popped and the element itself. Returns nil on timeout.

## Basic Usage

### Blocking Pop with Timeout

Open one Redis client and block on a list:

```bash
redis-cli BLPOP myqueue 10
```

In a second client, push an element:

```bash
redis-cli RPUSH myqueue "hello"
```

The first client immediately receives:

```text
1) "myqueue"
2) "hello"
```

### Timeout Expiry

If no element arrives before the timeout, BLPOP returns nil:

```bash
redis-cli BLPOP emptylist 2
```

After 2 seconds:

```text
(nil)
(2.00s)
```

### Immediate Return When List Has Data

If the list already has elements when BLPOP is called, it returns immediately without blocking:

```bash
redis-cli RPUSH jobs "job:1" "job:2"
redis-cli BLPOP jobs 5
```

```text
1) "jobs"
2) "job:1"
```

## Priority Queue Pattern

BLPOP checks keys left to right and pops from the first non-empty one:

```bash
redis-cli RPUSH queue:low "task:low"
redis-cli BLPOP queue:critical queue:high queue:normal queue:low 5
```

```text
1) "queue:low"
2) "task:low"
```

Add a high-priority item and it takes precedence:

```bash
redis-cli RPUSH queue:critical "task:urgent"
redis-cli BLPOP queue:critical queue:high queue:normal queue:low 5
```

```text
1) "queue:critical"
2) "task:urgent"
```

## Worker Queue Consumer in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def worker_loop():
    print("Worker started, waiting for jobs...")
    while True:
        # Block indefinitely until a job arrives
        result = r.blpop('jobqueue', timeout=0)
        if result:
            queue_name, job_data = result
            print(f"Processing job from {queue_name}: {job_data}")
            process_job(job_data)

def process_job(data):
    # Simulate job processing
    print(f"Job done: {data}")

worker_loop()
```

## Worker Queue Consumer in Node.js

```javascript
const redis = require('redis');

const client = redis.createClient();

async function workerLoop() {
  await client.connect();
  console.log('Worker started, waiting for jobs...');

  while (true) {
    const result = await client.blPop('jobqueue', 0);
    if (result) {
      console.log(`Processing job from ${result.key}: ${result.element}`);
      await processJob(result.element);
    }
  }
}

async function processJob(data) {
  console.log(`Job done: ${data}`);
}

workerLoop().catch(console.error);
```

## BRPOP Example - Stack-Based Consumption

BRPOP pops from the right end (tail), which is useful for stack-like patterns:

```bash
redis-cli RPUSH stack "bottom" "middle" "top"
redis-cli BRPOP stack 5
```

```text
1) "stack"
2) "top"
```

## Handling Multiple Concurrent Consumers

When multiple clients are blocked on the same key, Redis uses FIFO ordering - the first client to call BLPOP gets served first when a new element arrives. This ensures fair work distribution:

```bash
# Consumer 1 (started first)
redis-cli BLPOP shared_queue 0

# Consumer 2 (started second)
redis-cli BLPOP shared_queue 0

# Producer pushes one item - Consumer 1 gets it
redis-cli RPUSH shared_queue "task:1"
```

## Periodic Housekeeping with Short Timeout

Use a short timeout to allow workers to perform periodic maintenance between pops:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def worker_with_housekeeping():
    while True:
        result = r.blpop('workqueue', timeout=5)
        if result:
            _, job = result
            process_job(job)
        else:
            # Timeout expired - do housekeeping
            flush_metrics()
            check_health()

def flush_metrics():
    print("Flushing metrics...")

def check_health():
    print("Health check passed")

def process_job(job):
    print(f"Processing: {job}")
```

## Summary

`BLPOP` and `BRPOP` transform Redis lists into efficient event-driven queues by blocking consumers until work is available, eliminating polling overhead. They support priority ordering through multiple-key syntax and return the key name alongside the value so consumers know which queue served them. Use `timeout=0` for indefinitely blocking workers and short timeouts when periodic housekeeping is needed between jobs.
