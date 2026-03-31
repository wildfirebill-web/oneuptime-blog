# How to Use Mongoose with Transactions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Transaction, ACID, Node.js

Description: Learn how to use MongoDB multi-document ACID transactions with Mongoose to safely update multiple documents as an all-or-nothing operation.

---

## Prerequisites for Transactions

MongoDB transactions require a replica set or sharded cluster. A standalone mongod does not support transactions. For local development, start mongod as a single-node replica set:

```bash
mongod --replSet rs0 --bind_ip 127.0.0.1
mongosh --eval "rs.initiate()"
```

## Basic Transaction Pattern with Mongoose

```javascript
const mongoose = require('mongoose');

async function transferFunds(fromId, toId, amount) {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    // All operations pass the session
    await Account.findByIdAndUpdate(
      fromId,
      { $inc: { balance: -amount } },
      { session, runValidators: true }
    );

    await Account.findByIdAndUpdate(
      toId,
      { $inc: { balance: amount } },
      { session }
    );

    await session.commitTransaction();
  } catch (err) {
    await session.abortTransaction();
    throw err;
  } finally {
    session.endSession();
  }
}
```

Every operation that should be part of the transaction must receive `{ session }` as an option.

## Using the Callback API (withTransaction)

The `withTransaction` callback handles commit and abort automatically, including retries on transient errors:

```javascript
async function createOrderWithInventoryUpdate(orderData) {
  const session = await mongoose.startSession();

  const result = await session.withTransaction(async () => {
    // Deduct inventory
    const product = await Product.findOneAndUpdate(
      { _id: orderData.productId, stock: { $gte: orderData.quantity } },
      { $inc: { stock: -orderData.quantity } },
      { session, new: true }
    );

    if (!product) {
      throw new Error('Insufficient stock');
    }

    // Create the order
    const [order] = await Order.create([{
      ...orderData,
      status: 'confirmed',
      reservedAt: new Date()
    }], { session });

    return order;
  });

  session.endSession();
  return result;
}
```

`withTransaction` automatically retries on `TransientTransactionError` and commits when the callback resolves.

## Creating Documents in a Transaction

Note that `Model.create()` in a transaction must receive `[doc]` (array) and `{ session }` as the second argument - Mongoose does not support passing a session in the first argument:

```javascript
const [newDoc] = await MyModel.create([docData], { session });
```

The array syntax is required when using sessions with `create()`.

## Reading Within a Transaction

Reads inside a transaction see the transaction's own writes before commit:

```javascript
const session = await mongoose.startSession();
session.startTransaction({ readConcern: { level: 'snapshot' } });

try {
  // This read sees the state at transaction start
  const user = await User.findById(userId).session(session);

  // Modify and update
  await User.findByIdAndUpdate(
    userId,
    { $inc: { loginCount: 1 } },
    { session }
  );

  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

## Transaction Error Handling

Handle specific error labels for intelligent retry logic:

```javascript
try {
  await session.commitTransaction();
} catch (err) {
  if (err.hasErrorLabel('UnknownTransactionCommitResult')) {
    // Commit may or may not have succeeded - retry commit
    await retryCommit(session);
  } else if (err.hasErrorLabel('TransientTransactionError')) {
    // Retry the entire transaction
    await session.abortTransaction();
  }
  throw err;
}
```

## Summary

Use Mongoose transactions by starting a session with `mongoose.startSession()`, passing `{ session }` to every operation in the transaction, and committing or aborting based on success or failure. The `withTransaction` callback API is preferred because it handles retries and cleanup automatically. Use `Model.create([data], { session })` for inserts within transactions. Transactions require a replica set and add latency, so use them only when you genuinely need atomicity across multiple documents or collections.
