# How to Use Transactions with Mongoose in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Node.js, Transaction, Database

Description: Learn how to use multi-document ACID transactions with Mongoose in Node.js, including session management, error handling, and retry logic.

---

Multi-document transactions in MongoDB let you execute multiple operations atomically. Mongoose provides first-class support for transactions via sessions introduced in MongoDB 4.0. This guide walks through the full lifecycle of a transaction using Mongoose.

## Prerequisites

- MongoDB 4.0+ replica set or sharded cluster (transactions require a replica set)
- Mongoose 5.2+
- Node.js 14+

## Basic Transaction with Mongoose

The standard pattern uses `mongoose.startSession()` followed by `session.withTransaction()`:

```javascript
const mongoose = require('mongoose');

const Order = mongoose.model('Order', new mongoose.Schema({
  userId: String,
  items: Array,
  total: Number
}));

const Inventory = mongoose.model('Inventory', new mongoose.Schema({
  productId: String,
  quantity: Number
}));

async function placeOrder(userId, cartItems) {
  const session = await mongoose.startSession();

  try {
    await session.withTransaction(async () => {
      // Deduct inventory
      for (const item of cartItems) {
        const result = await Inventory.findOneAndUpdate(
          { productId: item.productId, quantity: { $gte: item.qty } },
          { $inc: { quantity: -item.qty } },
          { session, new: true }
        );

        if (!result) {
          throw new Error(`Insufficient stock for ${item.productId}`);
        }
      }

      // Create order
      await Order.create([{
        userId,
        items: cartItems,
        total: cartItems.reduce((sum, i) => sum + i.price * i.qty, 0)
      }], { session });
    });

    console.log('Order placed successfully');
  } finally {
    session.endSession();
  }
}
```

`session.withTransaction()` automatically handles commit and abort, and retries transient errors.

## Manual Transaction Control

For more control, use explicit `startTransaction`, `commitTransaction`, and `abortTransaction`:

```javascript
async function transferFunds(fromId, toId, amount) {
  const session = await mongoose.startSession();
  session.startTransaction();

  try {
    await Account.findByIdAndUpdate(
      fromId,
      { $inc: { balance: -amount } },
      { session }
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

## Passing Sessions to Queries

Every Mongoose operation in a transaction must receive the session option. Missing it causes that operation to run outside the transaction:

```javascript
// Correct - session passed
await User.findById(id).session(session);

// Correct - session in options
await Log.create([{ action: 'purchase' }], { session });

// Wrong - no session, runs outside transaction
await Log.create([{ action: 'purchase' }]);
```

## Retry Logic for Transient Errors

MongoDB may throw `TransientTransactionError` for network issues or lock contention. `withTransaction` handles this automatically, but if you use manual control, implement retries:

```javascript
const MAX_RETRIES = 3;

async function withRetry(fn) {
  for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (err.hasErrorLabel('TransientTransactionError') && attempt < MAX_RETRIES - 1) {
        continue;
      }
      throw err;
    }
  }
}
```

## Read Concern and Write Concern

For strong guarantees, configure the transaction's read and write concern:

```javascript
session.startTransaction({
  readConcern: { level: 'snapshot' },
  writeConcern: { w: 'majority' }
});
```

`snapshot` read concern ensures all reads within the transaction see a consistent snapshot. `w: 'majority'` ensures writes are durable.

## Common Pitfalls

- Transactions time out after 60 seconds by default. Keep them short.
- Do not perform non-idempotent side effects (like sending emails) inside the transaction callback; they may run multiple times on retry.
- Transactions cannot span multiple databases in a sharded cluster unless using MongoDB 4.2+.

## Summary

Mongoose transactions follow a session-based model where every operation must carry the same session object. Using `withTransaction` is the safest approach because it handles retries and cleanup automatically. Always use `w: 'majority'` write concern and keep transactions as short as possible to avoid contention.

