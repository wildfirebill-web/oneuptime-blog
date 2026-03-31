# How to Perform CRUD Operations with the MongoDB Node.js Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Node.js, Driver, CRUD, Database

Description: Learn how to perform create, read, update, and delete operations with the official MongoDB Node.js driver using practical code examples.

---

## Overview

The official MongoDB Node.js driver provides a low-level API for interacting with MongoDB. Understanding how to perform CRUD operations directly with the driver gives you full control over queries and updates without an ORM layer.

## Prerequisites

Install the driver and connect to MongoDB before running these examples:

```bash
npm install mongodb
```

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
await client.connect();
const db = client.db('mydb');
const col = db.collection('users');
```

## Create (Insert)

Use `insertOne` to add a single document, or `insertMany` for multiple documents:

```javascript
// Insert one document
const result = await col.insertOne({ name: 'Alice', age: 30, email: 'alice@example.com' });
console.log('Inserted ID:', result.insertedId);

// Insert multiple documents
const manyResult = await col.insertMany([
  { name: 'Bob', age: 25 },
  { name: 'Carol', age: 35 }
]);
console.log('Inserted count:', manyResult.insertedCount);
```

## Read (Find)

Use `findOne` to retrieve a single document or `find` with a cursor for multiple:

```javascript
// Find one document
const user = await col.findOne({ name: 'Alice' });
console.log(user);

// Find multiple with filter and projection
const users = await col.find(
  { age: { $gte: 25 } },
  { projection: { name: 1, email: 1, _id: 0 } }
).toArray();
console.log(users);
```

## Update

Use `updateOne`, `updateMany`, or `replaceOne` to modify documents:

```javascript
// Update one document
await col.updateOne(
  { name: 'Alice' },
  { $set: { age: 31 }, $currentDate: { lastModified: true } }
);

// Update multiple documents
const updateResult = await col.updateMany(
  { age: { $lt: 30 } },
  { $inc: { age: 1 } }
);
console.log('Modified:', updateResult.modifiedCount);
```

## Delete

Use `deleteOne` or `deleteMany` to remove documents:

```javascript
// Delete one document
await col.deleteOne({ name: 'Bob' });

// Delete multiple documents
const delResult = await col.deleteMany({ age: { $gt: 60 } });
console.log('Deleted:', delResult.deletedCount);
```

## Upsert

The `upsert` option inserts a document if no match is found:

```javascript
await col.updateOne(
  { email: 'dave@example.com' },
  { $set: { name: 'Dave', age: 28 } },
  { upsert: true }
);
```

## Counting Documents

Use `countDocuments` for accurate counts with a filter:

```javascript
const count = await col.countDocuments({ age: { $gte: 18 } });
console.log('Adults:', count);
```

## Error Handling

Always wrap operations in try-catch blocks and close the client in a `finally` block:

```javascript
try {
  await col.insertOne({ name: 'Eve' });
} catch (err) {
  console.error('CRUD error:', err.message);
} finally {
  await client.close();
}
```

## Summary

The MongoDB Node.js driver exposes `insertOne`, `insertMany`, `findOne`, `find`, `updateOne`, `updateMany`, `deleteOne`, and `deleteMany` as the core CRUD operations. Using projections, filters, and `upsert` options gives you fine-grained control. Always handle errors and close connections when done to avoid resource leaks.
