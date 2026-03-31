# How to Fix MongoError: RetryableWriteError in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Retryable Write, Error, Replica Set, Transactions

Description: Learn what causes MongoError RetryableWriteError in MongoDB and how to configure retry behavior, handle transactions, and avoid duplicate writes.

---

## Understanding the Error

A `RetryableWriteError` is thrown when a write operation fails with a network error or a primary election, and the driver's automatic retry also fails. The error label `RetryableWriteError` is attached to the exception:

```text
MongoServerError: insert { insert: "orders" ... } :: caused by :: Connection reset by peer
    errorLabels: ['RetryableWriteError']
```

Retryable writes are enabled by default in MongoDB drivers 3.6+. They allow the driver to retry a write exactly once after a transient failure.

## How Retryable Writes Work

When a write fails with a retriable error (network issue, primary failover), the driver automatically retries once against the new primary. The server uses the `txnNumber` and `lsid` (logical session ID) to ensure the write is idempotent - it won't be applied twice even if both attempts reach the server.

```javascript
// Retryable writes are enabled by default
const client = new MongoClient(uri, { retryWrites: true });
```

## Cause 1: Retry Also Failed

If the automatic retry also encounters an error (e.g., the replica set is still electing a new primary), the error propagates to your application:

```javascript
async function insertWithRetry(collection, doc, maxAttempts = 3) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      return await collection.insertOne(doc);
    } catch (err) {
      const isRetryable = err.errorLabels &&
                          err.errorLabels.includes('RetryableWriteError');
      if (isRetryable && i < maxAttempts - 1) {
        await new Promise(r => setTimeout(r, 500 * (i + 1)));
        continue;
      }
      throw err;
    }
  }
}
```

## Cause 2: Standalone Instance

Retryable writes require a replica set or sharded cluster. On a standalone `mongod`, the driver disables retryable writes and may surface errors differently:

```javascript
// Explicitly disable if using standalone (not recommended for production)
const client = new MongoClient(uri, { retryWrites: false });
```

For production, always use a replica set - it provides retryable writes, failover, and redundancy.

## Cause 3: Operations Not Supported by Retryable Writes

Some operations cannot be retried automatically:

- Multi-document writes with `ordered: true` (partial execution)
- `mapReduce` with output to a collection
- Write operations inside a transaction that was already partially committed

For these, use explicit transactions with retry logic:

```javascript
async function runTransactionWithRetry(session, fn) {
  while (true) {
    try {
      await session.withTransaction(fn);
      return;
    } catch (err) {
      if (err.hasErrorLabel('TransientTransactionError')) {
        continue; // retry the whole transaction
      }
      throw err;
    }
  }
}

const session = client.startSession();
await runTransactionWithRetry(session, async () => {
  const db = client.db('mydb');
  await db.collection('inventory').updateOne(
    { item: 'widget', qty: { $gte: 1 } },
    { $inc: { qty: -1 } },
    { session }
  );
  await db.collection('orders').insertOne(
    { item: 'widget', status: 'confirmed' },
    { session }
  );
});
session.endSession();
```

## Checking Error Labels

```javascript
try {
  await collection.insertOne(doc);
} catch (err) {
  if (err.hasErrorLabel('RetryableWriteError')) {
    console.log('Retryable write error - may retry');
  } else if (err.hasErrorLabel('UnknownTransactionCommitResult')) {
    console.log('Transaction commit result unknown - check idempotency');
  }
}
```

## Summary

`RetryableWriteError` indicates a write that the driver could not successfully retry. Ensure you are using a replica set (required for retryable writes), implement application-level retry logic with exponential backoff for additional resilience, and use explicit transactions with `TransientTransactionError` retry loops for multi-document operations.
