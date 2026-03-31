# How to Use Redis as a Message Queue (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Message Queue, List, Background Job, Beginner, Worker

Description: A beginner's guide to using Redis lists as a simple, reliable message queue for background job processing in any application.

---

## What Is a Message Queue

A message queue lets your application send work to be processed later, outside the request-response cycle. Instead of sending an email synchronously during a user registration, you push a job to a queue and a background worker sends it asynchronously. This keeps your API fast and decoupled from slow operations.

## Redis Lists as a Queue

Redis lists are doubly-linked lists that support efficient push and pop from both ends. For a queue:

- Producers push jobs to the right (`RPUSH`)
- Consumers pop jobs from the left (`LPOP` or `BLPOP`)

This gives you a FIFO (First In, First Out) queue with no extra dependencies.

```bash
# Producer: push a job
RPUSH queue:emails '{"to":"alice@example.com","subject":"Welcome"}'

# Consumer: pop and process
LPOP queue:emails

# Check queue depth
LLEN queue:emails
```

## Blocking Pop with BLPOP

`LPOP` returns immediately - if the queue is empty, it returns nil and you must poll. `BLPOP` blocks until an item is available, which is much more efficient:

```bash
# Block for up to 30 seconds waiting for an item
BLPOP queue:emails 30
```

## Simple Worker in Python

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def send_email(job):
    print(f"Sending email to {job['to']}: {job['subject']}")
    # actual email sending logic here

def run_worker():
    print('Worker started, waiting for jobs...')
    while True:
        try:
            # BLPOP blocks up to 5 seconds, returns (key, value) tuple
            result = r.blpop('queue:emails', timeout=5)
            if result:
                queue_name, raw_job = result
                job = json.loads(raw_job)
                print(f'Processing job: {job}')
                send_email(job)
        except Exception as e:
            print(f'Error processing job: {e}')
            time.sleep(1)  # brief pause before retrying

if __name__ == '__main__':
    run_worker()
```

## Producer in Python

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def enqueue_email(to, subject, body):
    job = json.dumps({
        'to': to,
        'subject': subject,
        'body': body,
        'enqueued_at': int(__import__('time').time())
    })
    r.rpush('queue:emails', job)
    print(f'Enqueued email to {to}')

# Usage
enqueue_email('alice@example.com', 'Welcome!', 'Thanks for signing up.')
enqueue_email('bob@example.com', 'Password Reset', 'Click here to reset.')
```

## Worker in Node.js

```javascript
const redis = require('redis');
const client = redis.createClient();

async function processEmailJob(job) {
  console.log(`Sending email to ${job.to}: ${job.subject}`);
  // actual sending logic
}

async function runWorker() {
  await client.connect();
  console.log('Worker started');

  while (true) {
    try {
      const result = await client.blPop('queue:emails', 5);
      if (result) {
        const job = JSON.parse(result.element);
        await processEmailJob(job);
      }
    } catch (err) {
      console.error('Worker error:', err);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}

runWorker();
```

## Multiple Queues with Priority

Use separate lists for different priority levels, and have workers check high-priority queues first:

```python
def run_priority_worker():
    queues = ['queue:critical', 'queue:high', 'queue:normal']
    while True:
        # BLPOP checks queues left to right, picks first available
        result = r.blpop(queues, timeout=5)
        if result:
            queue_name, raw_job = result
            job = json.loads(raw_job)
            process_job(queue_name, job)
```

## Failed Job Handling

Move failed jobs to a dead-letter queue for inspection:

```python
def safe_process(queue):
    result = r.blpop(queue, timeout=5)
    if not result:
        return
    queue_name, raw_job = result
    job = json.loads(raw_job)
    try:
        process_job(job)
    except Exception as e:
        print(f'Job failed: {e}')
        # Store failed job with error info
        failed = json.dumps({**job, 'error': str(e), 'failed_at': int(time.time())})
        r.rpush('queue:dead', failed)
```

## Monitoring Queue Depth

```bash
# Check how many jobs are waiting
LLEN queue:emails
LLEN queue:critical
LLEN queue:normal

# Peek at next job without removing it
LINDEX queue:emails 0
```

## Summary

Redis lists provide a simple, dependency-free message queue using `RPUSH` to enqueue jobs and `BLPOP` to process them efficiently without polling. This pattern is ideal for small to medium workloads and supports multiple queues with priority ordering. For production use cases with retry logic, job scheduling, and monitoring, consider libraries like BullMQ (Node.js) or Celery (Python) that build on top of Redis.
