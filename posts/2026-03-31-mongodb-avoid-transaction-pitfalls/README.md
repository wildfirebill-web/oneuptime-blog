# How to Avoid Common Transaction Pitfalls in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Transaction, Best Practice, ACID, Performance

Description: Avoid the most common MongoDB transaction mistakes including long transactions, missing retries, DDL inside transactions, and incorrect error handling patterns.

---

MongoDB transactions are powerful, but several common mistakes can cause data inconsistencies, poor performance, or silent failures. This guide covers the most frequent transaction pitfalls and how to avoid each one.

## Pitfall 1 - Not Retrying on TransientTransactionError

MongoDB can return a `TransientTransactionError` for temporary conditions: network interruptions, write conflicts, or primary elections. If you do not retry, you lose the operation entirely.

```javascript
// BAD: no retry logic
try {
  session.startTransaction()
  await db.collection('orders').insertOne({ ... }, { session })
  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
  throw err  // fails permanently on transient errors
}

// GOOD: use withTransaction (handles retries automatically)
await session.withTransaction(async (session) => {
  await db.collection('orders').insertOne({ ... }, { session })
})
```

## Pitfall 2 - Ignoring UnknownTransactionCommitResult

If `commitTransaction()` throws with `UnknownTransactionCommitResult`, the commit may or may not have succeeded. Never abort and restart - retry the commit:

```javascript
// BAD: aborting after unknown commit result causes duplicate work
} catch (err) {
  await session.abortTransaction()  // WRONG: commit may have succeeded
}

// GOOD: retry commit on UnknownTransactionCommitResult
async function commitWithRetry(session) {
  while (true) {
    try {
      await session.commitTransaction()
      return
    } catch (err) {
      if (err.errorLabels?.includes('UnknownTransactionCommitResult')) {
        continue  // retry commit only
      }
      throw err
    }
  }
}
```

## Pitfall 3 - DDL Operations Inside Transactions

You cannot create collections, create indexes, or drop collections inside a transaction in most MongoDB versions. These operations will fail or cause unexpected behavior:

```javascript
// BAD: DDL inside transaction
await session.withTransaction(async (session) => {
  await db.createCollection('newCollection')  // fails inside transaction
  await db.collection('newCollection').insertOne({ ... }, { session })
})

// GOOD: create collection before starting transaction
await db.createCollection('newCollection')
await session.withTransaction(async (session) => {
  await db.collection('newCollection').insertOne({ ... }, { session })
})
```

## Pitfall 4 - Long-Running Transactions

Holding a transaction open for more than a few seconds causes lock contention. Never do slow I/O, external API calls, or user-facing delays inside a transaction:

```javascript
// BAD: slow external call inside transaction
await session.withTransaction(async (session) => {
  const user = await db.collection('users').findOne({ _id: id }, { session })
  const risk = await externalRiskAPI(user)   // could take seconds
  await db.collection('decisions').insertOne({ userId: id, risk }, { session })
})

// GOOD: move I/O outside transaction
const user = await db.collection('users').findOne({ _id: id })
const risk = await externalRiskAPI(user)
await session.withTransaction(async (session) => {
  await db.collection('decisions').insertOne({ userId: id, risk }, { session })
})
```

## Pitfall 5 - Not Passing the Session to Every Operation

If you forget to pass `{ session }` to any operation, that operation runs outside the transaction and cannot be rolled back:

```javascript
// BAD: missing session on second operation
await session.withTransaction(async (session) => {
  await db.collection('inventory').updateOne(
    { sku: 'X' }, { $inc: { qty: -1 } }, { session }
  )
  // This insert is NOT in the transaction - it runs immediately and is permanent
  await db.collection('orders').insertOne({ sku: 'X' })  // missing session!
})

// GOOD: always include session
await session.withTransaction(async (session) => {
  await db.collection('inventory').updateOne(
    { sku: 'X' }, { $inc: { qty: -1 } }, { session }
  )
  await db.collection('orders').insertOne({ sku: 'X' }, { session })
})
```

## Pitfall 6 - Overusing Transactions

Not every operation needs a transaction. Single-document operations in MongoDB are always atomic. Using transactions for single-document updates adds unnecessary overhead:

```javascript
// UNNECESSARY: transaction for a single-document update
await session.withTransaction(async (session) => {
  await db.collection('users').updateOne(
    { _id: userId },
    { $set: { lastLogin: new Date() } },
    { session }
  )
})

// CORRECT: single-document ops are atomic without transactions
await db.collection('users').updateOne(
  { _id: userId },
  { $set: { lastLogin: new Date() } }
)
```

## Pitfall 7 - Not Ending Sessions

Leaking sessions wastes server resources. Always end sessions in a `finally` block:

```javascript
const session = client.startSession()
try {
  await session.withTransaction(async (s) => { /* ... */ })
} finally {
  await session.endSession()  // always runs, even on error
}
```

## Pitfall 8 - Using Transactions on Standalone MongoDB

Transactions require a replica set. A standalone `mongod` instance does not support them:

```bash
# Convert standalone to single-node replica set
mongod --replSet rs0 --dbpath /data/db
```

```javascript
rs.initiate()
```

## Summary

The most impactful MongoDB transaction mistakes are: not retrying on transient errors, incorrectly handling `UnknownTransactionCommitResult`, running DDL inside transactions, performing slow I/O within transactions, and forgetting to pass the session to operations. Use the `withTransaction()` callback API to automatically handle retries and cleanup, and reserve transactions only for operations that genuinely require multi-document atomicity.
