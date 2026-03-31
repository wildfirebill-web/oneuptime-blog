# How to Use Transactions with the MongoDB Node.js Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Node.js, Transaction, ACID, Driver

Description: Learn how to implement multi-document ACID transactions in MongoDB using the official Node.js driver with session management and error handling.

---

## Overview

MongoDB supports multi-document ACID transactions on replica sets and sharded clusters. The Node.js driver implements transactions via `ClientSession`, allowing you to group multiple operations into an atomic unit.

## Prerequisites

Transactions require MongoDB 4.0+ with a replica set. For local development, start a single-node replica set:

```bash
mongod --replSet rs0 --port 27017 --dbpath /data/db
# Then in mongo shell:
# rs.initiate()
```

## Basic Transaction Pattern

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
await client.connect();

const session = client.startSession();
try {
  session.startTransaction({
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' }
  });

  const accounts = client.db('bank').collection('accounts');

  await accounts.updateOne(
    { _id: 'alice' },
    { $inc: { balance: -100 } },
    { session }
  );

  await accounts.updateOne(
    { _id: 'bob' },
    { $inc: { balance: 100 } },
    { session }
  );

  await session.commitTransaction();
  console.log('Transaction committed');
} catch (err) {
  await session.abortTransaction();
  console.error('Transaction aborted:', err.message);
} finally {
  await session.endSession();
}
```

## Using the withTransaction Helper

The `withTransaction` helper automatically commits or retries on transient errors:

```javascript
async function transfer(client, fromId, toId, amount) {
  const session = client.startSession();
  const accounts = client.db('bank').collection('accounts');

  try {
    await session.withTransaction(async () => {
      const from = await accounts.findOne({ _id: fromId }, { session });
      if (from.balance < amount) throw new Error('Insufficient funds');

      await accounts.updateOne({ _id: fromId }, { $inc: { balance: -amount } }, { session });
      await accounts.updateOne({ _id: toId }, { $inc: { balance: amount } }, { session });
    });
  } finally {
    await session.endSession();
  }
}
```

## Handling Transient Transaction Errors

Transient errors (like write conflicts) should trigger a retry of the entire transaction:

```javascript
const { MongoError } = require('mongodb');

async function runWithRetry(txnFn, client) {
  const session = client.startSession();
  try {
    let committed = false;
    while (!committed) {
      try {
        session.startTransaction();
        await txnFn(session);
        await session.commitTransaction();
        committed = true;
      } catch (err) {
        await session.abortTransaction();
        if (!err.hasErrorLabel('TransientTransactionError')) throw err;
        console.log('Retrying transient transaction error...');
      }
    }
  } finally {
    await session.endSession();
  }
}
```

## Transaction Options

```javascript
session.startTransaction({
  readConcern: { level: 'snapshot' },  // consistent point-in-time view
  writeConcern: { w: 'majority' },     // wait for majority write acknowledgment
  maxCommitTimeMS: 5000                // abort if commit takes too long
});
```

## Summary

MongoDB transactions in the Node.js driver use `ClientSession` with `startTransaction`, `commitTransaction`, and `abortTransaction`. The `withTransaction` helper simplifies the pattern by handling retries on transient errors automatically. Always pass the `session` object to every operation inside a transaction and end the session in a `finally` block.
