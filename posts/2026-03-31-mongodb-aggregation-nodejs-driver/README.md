# How to Use Aggregation Pipelines with the MongoDB Node.js Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Node.js, Aggregation, Pipeline, Driver

Description: Learn how to build and execute aggregation pipelines with the MongoDB Node.js driver to group, filter, and transform documents efficiently.

---

## Overview

MongoDB aggregation pipelines process documents through a series of stages to compute summaries, reshape data, and perform analytics. The Node.js driver exposes the `aggregate` method on any collection.

## Basic Aggregation

Run a pipeline using `collection.aggregate()`, which returns a cursor:

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
await client.connect();
const col = client.db('shop').collection('orders');

const results = await col.aggregate([
  { $match: { status: 'completed' } },
  { $group: { _id: '$customerId', total: { $sum: '$amount' } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
]).toArray();

console.log(results);
```

## Common Pipeline Stages

```text
$match   - Filter documents (like WHERE in SQL)
$group   - Group by a field and compute accumulators
$project - Reshape documents, include/exclude fields
$sort    - Sort documents by one or more fields
$limit   - Return only N documents
$skip    - Skip N documents (use with $sort for pagination)
$unwind  - Deconstruct array fields into individual documents
$lookup  - Left outer join with another collection
$addFields - Add computed fields without replacing document
```

## Joining Collections with $lookup

```javascript
const invoices = await col.aggregate([
  { $match: { status: 'completed' } },
  {
    $lookup: {
      from: 'customers',
      localField: 'customerId',
      foreignField: '_id',
      as: 'customerInfo'
    }
  },
  { $unwind: '$customerInfo' },
  {
    $project: {
      orderId: 1,
      amount: 1,
      customerName: '$customerInfo.name'
    }
  }
]).toArray();
```

## Using $unwind with Arrays

```javascript
const tags = await col.aggregate([
  { $unwind: '$tags' },
  { $group: { _id: '$tags', count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]).toArray();
```

## Allowing Disk Use for Large Pipelines

For pipelines that exceed the 100 MB memory limit, enable disk use:

```javascript
const largeResult = await col.aggregate(
  [
    { $group: { _id: '$category', total: { $sum: '$amount' } } },
    { $sort: { total: -1 } }
  ],
  { allowDiskUse: true }
).toArray();
```

## Iterating with a Cursor

For large result sets, iterate the cursor instead of loading all into memory:

```javascript
const cursor = col.aggregate([
  { $match: { year: 2025 } },
  { $project: { _id: 0, month: 1, revenue: 1 } }
]);

for await (const doc of cursor) {
  process.stdout.write(JSON.stringify(doc) + '\n');
}
```

## Summary

The MongoDB Node.js driver's `aggregate` method accepts an array of pipeline stages and returns a cursor. Use `$match`, `$group`, `$lookup`, and `$project` to build powerful data transformations. For large datasets, enable `allowDiskUse` and iterate results with an async cursor to avoid memory issues.
