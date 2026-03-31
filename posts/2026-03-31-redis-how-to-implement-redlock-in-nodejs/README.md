# How to Implement Redlock in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redlock, Distributed Locking, Node.js, Concurrency

Description: Learn how to implement the Redlock distributed locking algorithm in Node.js using the redlock npm package for fault-tolerant distributed mutual exclusion.

---

## What Is Redlock

Redlock is the Redis distributed locking algorithm that acquires locks on N independent Redis instances. A lock is granted when a majority (N/2 + 1) of nodes grant it within the lock TTL window. This prevents a single Redis node failure from causing two processes to hold the same lock simultaneously.

## Installing the redlock Package

```bash
npm install redlock ioredis
# For TypeScript:
npm install redlock ioredis
# redlock includes TypeScript types
```

## Basic Redlock Usage

```javascript
const { default: Redlock } = require('redlock');
const Redis = require('ioredis');

// Connect to multiple independent Redis instances
const redis1 = new Redis({ host: 'redis-1.example.com', port: 6379 });
const redis2 = new Redis({ host: 'redis-2.example.com', port: 6379 });
const redis3 = new Redis({ host: 'redis-3.example.com', port: 6379 });

const redlock = new Redlock([redis1, redis2, redis3], {
  // Retry configuration
  retryCount: 3,
  retryDelay: 200,   // ms between retries
  retryJitter: 100,  // random jitter to prevent thundering herd

  // Drift factor: adds (ttl * driftFactor) as safety margin
  driftFactor: 0.01,

  // Automatic extension threshold
  automaticExtensionThreshold: 500 // ms before expiry to extend
});

async function processOrder(orderId) {
  let lock;
  try {
    // Acquire lock for 30 seconds
    lock = await redlock.acquire([`lock:order:${orderId}`], 30000);
    console.log(`Lock acquired for order ${orderId}`);

    // Your critical section
    await processPayment(orderId);
    await updateInventory(orderId);

    console.log(`Order ${orderId} processed successfully`);
  } catch (err) {
    if (err.name === 'ExecutionError') {
      console.error(`Could not acquire lock for order ${orderId}: ${err.message}`);
    } else {
      throw err;
    }
  } finally {
    if (lock) {
      await lock.release();
    }
  }
}
```

## TypeScript Usage

```typescript
import Redlock, { Lock } from 'redlock';
import Redis from 'ioredis';

const redis = new Redis({ host: 'localhost', port: 6379 });
const redlock = new Redlock([redis], {
  retryCount: 3,
  retryDelay: 200,
});

async function withLock<T>(
  resource: string,
  ttlMs: number,
  fn: () => Promise<T>
): Promise<T> {
  const lock: Lock = await redlock.acquire([resource], ttlMs);
  try {
    return await fn();
  } finally {
    await lock.release();
  }
}

// Usage
const result = await withLock('lock:user:42', 10000, async () => {
  const user = await db.getUser(42);
  user.balance -= 100;
  await db.saveUser(user);
  return user;
});
```

## Auto-Extending Locks for Long Operations

For operations that may exceed the initial TTL, use the using() method which automatically extends the lock:

```javascript
const { default: Redlock } = require('redlock');
const Redis = require('ioredis');

const redis = new Redis();
const redlock = new Redlock([redis], {
  automaticExtensionThreshold: 500 // ms before expiry to extend
});

async function longRunningJob(jobId) {
  // using() automatically extends the lock before it expires
  const result = await redlock.using(
    [`lock:job:${jobId}`],
    30000,  // initial TTL
    async (signal) => {
      // signal.aborted is set if extension fails
      for (let step = 0; step < 10; step++) {
        if (signal.aborted) {
          throw new Error('Lock extension failed - aborting job');
        }

        await processStep(jobId, step);
        console.log(`Step ${step} complete`);
      }

      return { jobId, status: 'complete' };
    }
  );

  return result;
}
```

## Single Redis Instance Lock (Simpler)

For development or when distributed consensus is not required:

```javascript
const Redis = require('ioredis');
const crypto = require('crypto');

const redis = new Redis({ host: 'localhost', port: 6379 });

const ACQUIRE_SCRIPT = `
  if redis.call('exists', KEYS[1]) == 0 then
    return redis.call('set', KEYS[1], ARGV[1], 'px', ARGV[2])
  end
  return nil
`;

const RELEASE_SCRIPT = `
  if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
  end
  return 0
`;

class SimpleLock {
  constructor(redisClient) {
    this.redis = redisClient;
  }

  async acquire(resource, ttlMs) {
    const lockValue = crypto.randomUUID();
    const result = await this.redis.eval(
      ACQUIRE_SCRIPT, 1, resource, lockValue, String(ttlMs)
    );

    if (result !== 'OK') {
      throw new Error(`Could not acquire lock for ${resource}`);
    }

    return { resource, value: lockValue };
  }

  async release(lock) {
    await this.redis.eval(RELEASE_SCRIPT, 1, lock.resource, lock.value);
  }

  async withLock(resource, ttlMs, fn) {
    const lock = await this.acquire(resource, ttlMs);
    try {
      return await fn();
    } finally {
      await this.release(lock);
    }
  }
}

// Usage
const locker = new SimpleLock(redis);

await locker.withLock('resource:42', 5000, async () => {
  console.log('Inside critical section');
  await someAsyncWork();
});
```

## Handling Lock Expiration Gracefully

```javascript
const { default: Redlock, ExecutionError, ResourceLockedError } = require('redlock');
const Redis = require('ioredis');

const redis = new Redis();
const redlock = new Redlock([redis], { retryCount: 0 }); // No retry

async function tryAcquireLock(resource, ttlMs) {
  try {
    return await redlock.acquire([resource], ttlMs);
  } catch (err) {
    if (err instanceof ExecutionError) {
      // Lock is held by another process
      return null;
    }
    throw err;
  }
}

async function processIfAvailable(jobId) {
  const lock = await tryAcquireLock(`lock:job:${jobId}`, 30000);

  if (!lock) {
    console.log(`Job ${jobId} is already being processed by another worker`);
    return { skipped: true };
  }

  try {
    const result = await doWork(jobId);
    return { processed: true, result };
  } finally {
    await lock.release();
  }
}
```

## Testing Lock Contention

```javascript
const { default: Redlock, ExecutionError } = require('redlock');
const Redis = require('ioredis');

async function testLockContention() {
  const redis = new Redis();
  const redlock = new Redlock([redis], { retryCount: 0 });

  let worker1Ran = false;
  let worker2Ran = false;

  async function worker(id) {
    let lock;
    try {
      lock = await redlock.acquire(['lock:test-resource'], 5000);
      console.log(`Worker ${id} acquired lock`);
      if (id === 1) worker1Ran = true;
      if (id === 2) worker2Ran = true;
      await new Promise(resolve => setTimeout(resolve, 100));
    } catch (err) {
      if (err instanceof ExecutionError) {
        console.log(`Worker ${id} could not acquire lock (expected)`);
      }
    } finally {
      if (lock) await lock.release();
    }
  }

  // Run both workers concurrently
  await Promise.all([worker(1), worker(2)]);

  // One worker should have run, the other should have been blocked
  const bothRan = worker1Ran && worker2Ran;
  console.log(`Both workers ran: ${bothRan}`); // Should be false with retryCount=0

  await redis.quit();
}

testLockContention();
```

## Summary

Implementing Redlock in Node.js is straightforward with the `redlock` npm package. Pass multiple independent Redis client instances to Redlock for proper distributed consensus, configure retryCount and retryDelay for your application's contention tolerance, and use the using() method for long-running operations that require automatic lock extension. Always release locks in a finally block to prevent deadlocks, and use the atomic Lua release script to ensure only the lock owner can release it. For development with a single Redis instance, a simpler SET NX + Lua release pattern is sufficient.
