# How to Implement Custom Retry Logic for MongoDB Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Retry, Error Handling, Resilience, Driver

Description: Learn how to implement custom retry logic with exponential backoff for MongoDB operations to handle transient failures beyond built-in driver retries.

---

## Why Custom Retry Logic?

MongoDB drivers provide built-in retryable reads and writes, but they only retry once and only for specific error codes. For production systems that need higher resilience - such as tolerating longer network partitions, transient Atlas overload errors, or multi-step operations - you need custom retry logic with exponential backoff and jitter.

## Basic Retry Wrapper in Node.js

```javascript
async function withRetry(operation, { maxAttempts = 3, baseDelayMs = 100 } = {}) {
  let lastError;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (err) {
      lastError = err;
      const isRetryable = isTransientError(err);
      if (!isRetryable || attempt === maxAttempts) {
        throw err;
      }
      const delay = baseDelayMs * Math.pow(2, attempt - 1) + Math.random() * 50;
      console.warn(`Attempt ${attempt} failed, retrying in ${delay.toFixed(0)}ms`);
      await new Promise((res) => setTimeout(res, delay));
    }
  }
  throw lastError;
}

function isTransientError(err) {
  const transientCodes = new Set([6, 7, 89, 91, 189, 9001, 10107, 11600, 13435]);
  return transientCodes.has(err?.code) || err?.hasErrorLabel?.('RetryableWriteError');
}
```

## Using the Retry Wrapper

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient('mongodb://localhost:27017', { retryWrites: false });

async function updateOrderStatus(orderId, status) {
  return withRetry(
    () =>
      client
        .db('shop')
        .collection('orders')
        .updateOne({ _id: orderId }, { $set: { status, updatedAt: new Date() } }),
    { maxAttempts: 5, baseDelayMs: 200 }
  );
}
```

## Python Implementation with Tenacity

```python
from pymongo import MongoClient
from pymongo.errors import AutoReconnect, NetworkTimeout, ConnectionFailure
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

client = MongoClient("mongodb://localhost:27017")
db = client["shop"]

TRANSIENT_ERRORS = (AutoReconnect, NetworkTimeout, ConnectionFailure)

@retry(
    retry=retry_if_exception_type(TRANSIENT_ERRORS),
    wait=wait_exponential(multiplier=0.1, min=0.1, max=5),
    stop=stop_after_attempt(5),
)
def update_order_status(order_id, status):
    db.orders.update_one(
        {"_id": order_id},
        {"$set": {"status": status}}
    )
```

## Retry Logic for Transactions

Transactions require special handling - they can fail with `TransientTransactionError` or `UnknownTransactionCommitResult` labels:

```javascript
async function runTransactionWithRetry(client, txnFunc) {
  const session = client.startSession();
  try {
    await session.withTransaction(async () => {
      await txnFunc(session);
    }, {
      readPreference: 'primary',
      readConcern: { level: 'local' },
      writeConcern: { w: 'majority' },
    });
  } finally {
    await session.endSession();
  }
}

// withTransaction() handles TransientTransactionError retries automatically
await runTransactionWithRetry(client, async (session) => {
  const orders = client.db('shop').collection('orders');
  const inventory = client.db('shop').collection('inventory');
  await orders.insertOne({ item: 'widget', qty: 5 }, { session });
  await inventory.updateOne({ item: 'widget' }, { $inc: { qty: -5 } }, { session });
});
```

## Adding Circuit Breaker Pattern

For high-traffic systems, combine retries with a circuit breaker to avoid cascading failures:

```javascript
class CircuitBreaker {
  constructor({ threshold = 5, timeout = 30000 } = {}) {
    this.failures = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED';
    this.nextAttempt = null;
  }

  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) throw new Error('Circuit open');
      this.state = 'HALF_OPEN';
    }
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() { this.failures = 0; this.state = 'CLOSED'; }
  onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

## Summary

Custom retry logic fills the gap between driver-level retries and fully resilient systems. Use exponential backoff with jitter to avoid thundering-herd problems, limit retries to transient error codes only, and leverage MongoDB's built-in `withTransaction` for transaction retries. Combining retries with circuit breakers provides defense in depth for production MongoDB workloads.
