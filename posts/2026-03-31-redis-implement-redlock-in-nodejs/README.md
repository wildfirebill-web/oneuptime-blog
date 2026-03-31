# How to Implement Redlock in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redlock, Node.js, Distributed Locks, Concurrency, JavaScript

Description: Implement Redlock distributed locking in Node.js using the redlock npm package across multiple Redis instances for fault-tolerant exclusive resource access.

---

## What is Redlock

Redlock is an algorithm for distributed mutual exclusion using multiple independent Redis nodes. A lock is considered acquired if a majority of nodes (N/2 + 1) grant it within the lock's validity period. This ensures safety even when individual Redis nodes fail.

## Install Dependencies

```bash
npm install redlock ioredis
```

## Single Redis Instance Lock

For most applications, locking against a single Redis instance is sufficient:

```javascript
const Redlock = require("redlock");
const Redis = require("ioredis");

const redis = new Redis({ host: "localhost", port: 6379 });

const redlock = new Redlock([redis], {
  driftFactor: 0.01,
  retryCount: 10,
  retryDelay: 200,
  retryJitter: 200,
  automaticExtensionThreshold: 500,
});

async function processWithLock(resourceId) {
  let lock;
  try {
    lock = await redlock.acquire([`lock:${resourceId}`], 10000);
    console.log("Lock acquired");

    // Critical section
    await doWork(resourceId);

    console.log("Work completed");
  } catch (err) {
    if (err.name === "ExecutionError") {
      console.error("Could not acquire lock:", err.message);
    } else {
      throw err;
    }
  } finally {
    if (lock) {
      await lock.release();
      console.log("Lock released");
    }
  }
}

async function doWork(id) {
  await new Promise((r) => setTimeout(r, 500));
  console.log(`Processed resource ${id}`);
}

processWithLock("order:5001");
```

## Multi-Node Redlock for Fault Tolerance

```javascript
const Redlock = require("redlock");
const Redis = require("ioredis");

const nodes = [
  new Redis({ host: "redis-1", port: 6379 }),
  new Redis({ host: "redis-2", port: 6379 }),
  new Redis({ host: "redis-3", port: 6379 }),
];

const redlock = new Redlock(nodes, {
  driftFactor: 0.01,
  retryCount: 5,
  retryDelay: 300,
  retryJitter: 100,
  automaticExtensionThreshold: 500,
});

async function runWithDistributedLock(resource, ttlMs, work) {
  return await redlock.using([resource], ttlMs, async (signal) => {
    if (signal.aborted) {
      throw signal.error;
    }
    await work();
  });
}

runWithDistributedLock("lock:payment:order:5001", 15000, async () => {
  console.log("Processing payment for order 5001...");
  await new Promise((r) => setTimeout(r, 1000));
  console.log("Payment processed");
});
```

## Lock with Automatic Extension

The `using` method automatically extends the lock while your work is running:

```javascript
const redlock = new Redlock([redis], {
  automaticExtensionThreshold: 500,
});

async function longRunningTask(jobId) {
  await redlock.using([`job:lock:${jobId}`], 5000, async (signal) => {
    for (let step = 0; step < 10; step++) {
      if (signal.aborted) {
        throw new Error(`Lock lost during step ${step}: ${signal.error.message}`);
      }
      console.log(`Processing step ${step}`);
      await new Promise((r) => setTimeout(r, 800));
    }
  });
}
```

## Handling Lock Contention

```javascript
const { ExecutionError } = require("redlock");

async function tryAcquireWithTimeout(resource, ttlMs, timeoutMs = 5000) {
  const deadline = Date.now() + timeoutMs;
  const lock_redlock = new Redlock([redis], {
    retryCount: Math.floor(timeoutMs / 200),
    retryDelay: 200,
  });

  try {
    const lock = await lock_redlock.acquire([resource], ttlMs);
    return lock;
  } catch (err) {
    if (err instanceof ExecutionError) {
      console.error("Lock acquisition failed after retries");
      return null;
    }
    throw err;
  }
}
```

## Scoped Locking Pattern

```javascript
class InventoryService {
  constructor(redis) {
    this.redis = redis;
    this.redlock = new Redlock([redis], {
      retryCount: 5,
      retryDelay: 100,
    });
  }

  async reserveStock(productId, quantity) {
    const lockKey = `lock:inventory:${productId}`;
    return this.redlock.using([lockKey], 5000, async () => {
      const available = parseInt(await this.redis.get(`stock:${productId}`) || "0");
      if (available < quantity) {
        throw new Error(`Insufficient stock: ${available} < ${quantity}`);
      }
      await this.redis.decrby(`stock:${productId}`, quantity);
      return { reserved: quantity, remaining: available - quantity };
    });
  }
}

const svc = new InventoryService(redis);
svc.reserveStock("product:1001", 2)
  .then(console.log)
  .catch(console.error);
```

## TypeScript Example

```typescript
import Redlock, { ExecutionError } from "redlock";
import { Redis } from "ioredis";

const redis = new Redis();
const redlock = new Redlock([redis]);

async function withLock<T>(
  resource: string,
  ttl: number,
  fn: () => Promise<T>
): Promise<T> {
  return redlock.using([resource], ttl, fn);
}

await withLock("lock:user:101", 5000, async () => {
  // exclusive user update
});
```

## Summary

Node.js Redlock uses the `redlock` npm package backed by one or more `ioredis` clients. The `using` method handles lock acquisition, automatic extension, and release as a transaction around your work function. For fault tolerance in production, provide 3-5 independent Redis nodes and set the quorum to N/2 + 1. Always handle `ExecutionError` when the lock cannot be acquired within the retry budget.
