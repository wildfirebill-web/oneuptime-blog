# How to Test Aggregation Pipelines in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Testing, Pipeline, Jest

Description: Learn how to test MongoDB aggregation pipelines by running them against a real in-memory database with known seed data to verify correctness of each stage.

---

## Why Test Aggregation Pipelines?

Aggregation pipelines are complex - they combine multiple stages, use expressions, perform joins, and can produce unexpected results with edge-case data. Unit testing with mocks cannot verify that your pipeline logic is correct. Running pipelines against a real (in-memory) MongoDB instance with carefully crafted seed data is the most reliable way to test them.

## Setup

```bash
npm install --save-dev mongodb-memory-server mongodb jest
```

```javascript
// tests/aggregation.test.js
const { MongoMemoryServer } = require('mongodb-memory-server');
const { MongoClient } = require('mongodb');

let mongod, client, db;

beforeAll(async () => {
  mongod = await MongoMemoryServer.create();
  client = new MongoClient(mongod.getUri());
  await client.connect();
  db = client.db('testdb');
});

afterAll(async () => {
  await client.close();
  await mongod.stop();
});

beforeEach(async () => {
  const collections = await db.collections();
  for (const c of collections) await c.deleteMany({});
});
```

## Testing a $group Stage

```javascript
it('calculates total revenue per product category', async () => {
  const orders = db.collection('orders');
  await orders.insertMany([
    { category: 'electronics', amount: 500 },
    { category: 'electronics', amount: 300 },
    { category: 'clothing', amount: 150 },
    { category: 'clothing', amount: 75 },
  ]);

  const result = await orders.aggregate([
    { $group: { _id: '$category', totalRevenue: { $sum: '$amount' } } },
    { $sort: { _id: 1 } },
  ]).toArray();

  expect(result).toEqual([
    { _id: 'clothing', totalRevenue: 225 },
    { _id: 'electronics', totalRevenue: 800 },
  ]);
});
```

## Testing a $lookup Join

```javascript
it('enriches orders with user information via $lookup', async () => {
  const orders = db.collection('orders');
  const users = db.collection('users');

  const userId = new (require('mongodb').ObjectId)();
  await users.insertOne({ _id: userId, name: 'Alice' });
  await orders.insertOne({ userId, product: 'Laptop', amount: 1200 });

  const result = await orders.aggregate([
    {
      $lookup: {
        from: 'users',
        localField: 'userId',
        foreignField: '_id',
        as: 'user',
      },
    },
    { $unwind: '$user' },
    { $project: { product: 1, amount: 1, 'user.name': 1 } },
  ]).toArray();

  expect(result).toHaveLength(1);
  expect(result[0].user.name).toBe('Alice');
  expect(result[0].product).toBe('Laptop');
});
```

## Testing Edge Cases

```javascript
it('returns empty array when no documents match', async () => {
  const result = await db.collection('orders').aggregate([
    { $match: { category: 'nonexistent' } },
    { $group: { _id: '$category', total: { $sum: '$amount' } } },
  ]).toArray();

  expect(result).toEqual([]);
});

it('handles null and missing fields in $group', async () => {
  const orders = db.collection('orders');
  await orders.insertMany([
    { amount: 100 },            // missing category
    { category: null, amount: 50 },
  ]);

  const result = await orders.aggregate([
    { $group: { _id: '$category', total: { $sum: '$amount' } } },
  ]).toArray();

  expect(result).toHaveLength(1);
  expect(result[0]._id).toBeNull();
  expect(result[0].total).toBe(150);
});
```

## Extracting and Reusing Pipelines

For complex pipelines, extract them into named functions and test them independently:

```javascript
// src/pipelines/revenuePipeline.js
const getRevenuePipeline = ({ startDate, endDate }) => [
  { $match: { createdAt: { $gte: startDate, $lte: endDate } } },
  { $group: { _id: '$category', revenue: { $sum: '$amount' } } },
  { $sort: { revenue: -1 } },
];
module.exports = { getRevenuePipeline };
```

```javascript
it('filters revenue by date range', async () => {
  const { getRevenuePipeline } = require('../src/pipelines/revenuePipeline');
  const orders = db.collection('orders');
  await orders.insertMany([
    { category: 'A', amount: 100, createdAt: new Date('2026-01-15') },
    { category: 'B', amount: 200, createdAt: new Date('2026-02-20') },
  ]);
  const pipeline = getRevenuePipeline({
    startDate: new Date('2026-02-01'),
    endDate: new Date('2026-02-28'),
  });
  const result = await orders.aggregate(pipeline).toArray();
  expect(result).toHaveLength(1);
  expect(result[0]._id).toBe('B');
});
```

## Summary

Test aggregation pipelines against a real MongoDB instance (in-memory) with deterministic seed data. Write separate tests for normal cases, empty results, and missing/null field edge cases. Extract pipelines into standalone functions to make them both testable and reusable across your application.
