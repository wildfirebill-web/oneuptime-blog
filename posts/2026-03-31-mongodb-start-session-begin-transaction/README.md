# How to Start a Session and Begin a Transaction in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Transaction, Session, Multi-Document, ACID

Description: Learn how to start a client session and begin a multi-document transaction in MongoDB using Node.js, including session options and transaction settings.

---

MongoDB multi-document transactions require a client session as their foundation. A session provides a context for a series of operations, enabling atomicity across multiple documents and collections. This guide explains how to start a session, begin a transaction, and configure the relevant options using the Node.js driver.

## Prerequisites

Multi-document transactions require:
- MongoDB 4.0 or later for replica sets
- MongoDB 4.2 or later for sharded clusters
- A running replica set (standalone deployments do not support transactions)

## Starting a Session

A `ClientSession` is created from the `MongoClient` instance. The session tracks transaction state and routes operations to the correct server:

```javascript
const { MongoClient } = require('mongodb')

const client = new MongoClient('mongodb://localhost:27017')
await client.connect()

const session = client.startSession()
```

By default, sessions use causal consistency, ensuring that operations within the session respect write ordering. You can configure this:

```javascript
const session = client.startSession({
  causalConsistency: true,  // default: true
  defaultTransactionOptions: {
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority' },
    readPreference: 'primary'
  }
})
```

## Beginning a Transaction

After creating the session, call `startTransaction()` to open a transaction. All subsequent operations using this session will be part of the transaction:

```javascript
session.startTransaction({
  readConcern: { level: 'snapshot' },
  writeConcern: { w: 'majority' }
})
```

Transaction options specified in `startTransaction()` override the session defaults.

## Performing Operations Within the Transaction

Pass the session object to every operation that should participate in the transaction:

```javascript
const db = client.db('shop')

try {
  session.startTransaction()

  // Deduct inventory
  await db.collection('inventory').updateOne(
    { productId: 'SKU-001', quantity: { $gte: 1 } },
    { $inc: { quantity: -1 } },
    { session }
  )

  // Create order
  await db.collection('orders').insertOne(
    {
      productId: 'SKU-001',
      userId: 'user123',
      quantity: 1,
      status: 'pending',
      createdAt: new Date()
    },
    { session }
  )

  // Commit if all operations succeed
  await session.commitTransaction()
  console.log('Transaction committed successfully')

} catch (error) {
  // Abort on any error
  await session.abortTransaction()
  console.error('Transaction aborted:', error.message)

} finally {
  await session.endSession()
}
```

## Checking Transaction State

You can inspect the session state programmatically:

```javascript
console.log(session.inTransaction())     // true while transaction is active
console.log(session.id)                  // session identifier (LSID)
```

## Session Lifecycle

Always end the session after use, whether the transaction committed or aborted. Failing to end a session leaks server-side resources:

```javascript
const session = client.startSession()
try {
  session.startTransaction()
  // ... operations
  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
} finally {
  await session.endSession()  // always run this
}
```

## Using withSession for Automatic Cleanup

The `withSession` helper ensures the session is always ended, even if an unexpected error occurs:

```javascript
await client.withSession(async (session) => {
  session.startTransaction()
  try {
    await db.collection('accounts').updateOne(
      { _id: 'acct-A' },
      { $inc: { balance: -100 } },
      { session }
    )
    await db.collection('accounts').updateOne(
      { _id: 'acct-B' },
      { $inc: { balance: 100 } },
      { session }
    )
    await session.commitTransaction()
  } catch (err) {
    await session.abortTransaction()
    throw err
  }
})
```

## Summary

Starting a MongoDB transaction requires first creating a `ClientSession` via `client.startSession()`, then calling `session.startTransaction()`. Every read and write operation must receive the session object to be included in the transaction. Always end sessions in a `finally` block or use `withSession` to ensure clean session lifecycle management.
