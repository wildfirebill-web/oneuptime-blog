# How to Create and Manage Sessions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Session, Transaction, Driver, Consistency

Description: Learn how to create and manage client sessions in MongoDB to enable transactions, causal consistency, and retryable writes across multiple operations.

---

## Overview

A MongoDB client session is a logical context that groups related operations together. Sessions enable multi-document ACID transactions, causal consistency (read-your-own-writes guarantees), and retryable writes. Every MongoDB 4.0+ operation can be associated with a session.

## Creating a Session (Node.js Driver)

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(process.env.MONGODB_URI);
await client.connect();

// Start a new client session
const session = client.startSession();
```

Sessions are associated with the client that created them and are not shared across clients.

## Using a Session Without Transactions

You can pass a session to any operation to gain causal consistency and retryable writes without starting a full transaction:

```javascript
const db = client.db('shop');

// Read with session
const order = await db.collection('orders').findOne(
  { _id: orderId },
  { session }
);

// Write with session - subsequent reads will see this write
await db.collection('orders').updateOne(
  { _id: orderId },
  { $set: { status: 'processed' } },
  { session }
);
```

## Running a Transaction Within a Session

```javascript
async function transferFunds(fromId, toId, amount) {
  const session = client.startSession();

  try {
    session.startTransaction({
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
    });

    const accounts = client.db('bank').collection('accounts');

    await accounts.updateOne(
      { _id: fromId, balance: { $gte: amount } },
      { $inc: { balance: -amount } },
      { session }
    );

    await accounts.updateOne(
      { _id: toId },
      { $inc: { balance: amount } },
      { session }
    );

    await session.commitTransaction();
    return { success: true };
  } catch (err) {
    await session.abortTransaction();
    throw err;
  } finally {
    await session.endSession();
  }
}
```

## Using withTransaction for Automatic Retry

The `withTransaction` helper automatically retries the callback on transient errors:

```javascript
async function safeTransfer(fromId, toId, amount) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      const accounts = client.db('bank').collection('accounts');

      await accounts.updateOne(
        { _id: fromId, balance: { $gte: amount } },
        { $inc: { balance: -amount } },
        { session }
      );

      await accounts.updateOne(
        { _id: toId },
        { $inc: { balance: amount } },
        { session }
      );
    }, {
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
    });
  } finally {
    await session.endSession();
  }
}
```

## Checking Session State

```javascript
console.log(session.id);               // The session's LSID (logical session ID)
console.log(session.inTransaction());  // true if a transaction is in progress
```

## Session Lifetime Best Practices

- Always call `session.endSession()` in a `finally` block to release the session back to the pool.
- Do not hold a session open longer than needed - idle sessions consume server resources.
- Avoid sharing sessions across concurrent operations; each concurrent flow should use its own session.
- Sessions automatically expire on the server after 30 minutes of inactivity.

```javascript
// Always use try/finally to ensure cleanup
const session = client.startSession();
try {
  // ... your operations
} finally {
  await session.endSession();
}
```

## Summary

MongoDB sessions are the foundation for transactions and causal consistency. Create a session with `client.startSession()`, pass it to every operation within a logical unit of work, and always end it in a `finally` block. Use `withTransaction()` for automatic retry of transient errors. Sessions are lightweight, but idle sessions held open for long periods can impact server-side resource limits.
