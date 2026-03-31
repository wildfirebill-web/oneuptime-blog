# How to Test MongoDB Transactions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Transaction, Testing, Replica Set, Jest

Description: Learn how to write tests for MongoDB multi-document transactions, verifying atomicity, commit behavior, and rollback scenarios with a real MongoDB instance.

---

## Transactions Require a Replica Set

MongoDB multi-document transactions only work on replica sets or sharded clusters, not standalone instances. For testing, use `mongodb-memory-server` in replica set mode or Testcontainers with a replica set container.

## Setup with mongodb-memory-server Replica Set

```bash
npm install --save-dev mongodb-memory-server mongodb jest
```

```javascript
// tests/transactions.test.js
const { MongoMemoryReplSet } = require('mongodb-memory-server');
const { MongoClient } = require('mongodb');

let replSet, client, db;

beforeAll(async () => {
  replSet = await MongoMemoryReplSet.create({ replSet: { count: 1 } });
  client = new MongoClient(replSet.getUri());
  await client.connect();
  db = client.db('testdb');
}, 60000);

afterAll(async () => {
  await client.close();
  await replSet.stop();
});

beforeEach(async () => {
  await db.collection('accounts').deleteMany({});
  await db.collection('transfers').deleteMany({});
});
```

## Testing a Successful Transaction

```javascript
it('transfers funds atomically between accounts', async () => {
  const accounts = db.collection('accounts');
  const transfers = db.collection('transfers');

  await accounts.insertMany([
    { _id: 'acc-alice', balance: 1000 },
    { _id: 'acc-bob', balance: 500 },
  ]);

  const session = client.startSession();
  await session.withTransaction(async () => {
    await accounts.updateOne(
      { _id: 'acc-alice' },
      { $inc: { balance: -200 } },
      { session }
    );
    await accounts.updateOne(
      { _id: 'acc-bob' },
      { $inc: { balance: 200 } },
      { session }
    );
    await transfers.insertOne(
      { from: 'acc-alice', to: 'acc-bob', amount: 200, ts: new Date() },
      { session }
    );
  });
  await session.endSession();

  const alice = await accounts.findOne({ _id: 'acc-alice' });
  const bob = await accounts.findOne({ _id: 'acc-bob' });
  expect(alice.balance).toBe(800);
  expect(bob.balance).toBe(700);
  expect(await transfers.countDocuments()).toBe(1);
});
```

## Testing Transaction Rollback

```javascript
it('rolls back all changes when transaction aborts', async () => {
  const accounts = db.collection('accounts');

  await accounts.insertMany([
    { _id: 'acc-charlie', balance: 100 },
    { _id: 'acc-dave', balance: 100 },
  ]);

  const session = client.startSession();
  session.startTransaction();
  try {
    await accounts.updateOne(
      { _id: 'acc-charlie' },
      { $inc: { balance: -50 } },
      { session }
    );
    // Simulate a failure mid-transaction
    throw new Error('Simulated failure');
  } catch (_err) {
    await session.abortTransaction();
  } finally {
    await session.endSession();
  }

  // Both balances should be unchanged
  const charlie = await accounts.findOne({ _id: 'acc-charlie' });
  const dave = await accounts.findOne({ _id: 'acc-dave' });
  expect(charlie.balance).toBe(100);
  expect(dave.balance).toBe(100);
});
```

## Testing Write Conflict Detection

```javascript
it('detects a write conflict with two concurrent sessions', async () => {
  const docs = db.collection('docs');
  await docs.insertOne({ _id: 'shared', value: 0 });

  const session1 = client.startSession();
  const session2 = client.startSession();

  session1.startTransaction();
  session2.startTransaction();

  // session1 reads the document
  await docs.findOne({ _id: 'shared' }, { session: session1 });

  // session2 writes to the same document
  await docs.updateOne({ _id: 'shared' }, { $inc: { value: 1 } }, { session: session2 });
  await session2.commitTransaction();
  await session2.endSession();

  // session1 tries to write - should fail with WriteConflict
  await expect(
    docs.updateOne({ _id: 'shared' }, { $inc: { value: 10 } }, { session: session1 })
  ).rejects.toMatchObject({ code: 112 }); // WriteConflict

  await session1.abortTransaction();
  await session1.endSession();
});
```

## Summary

Test MongoDB transactions using `MongoMemoryReplSet` (not standalone). Write tests for the happy path (commit), the failure path (abort/rollback), and concurrent access scenarios (write conflict detection). Always use `withTransaction()` for production code to get automatic retry on `TransientTransactionError`, and verify atomicity by checking all affected collections after each test.
