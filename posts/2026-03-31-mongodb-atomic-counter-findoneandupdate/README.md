# How to Implement Atomic Counter with findOneAndUpdate in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atomic, Counter, FindOneAndUpdate, Concurrency

Description: Learn how to implement thread-safe atomic counters in MongoDB using findOneAndUpdate with the $inc operator for sequential ID generation.

---

## Why Atomic Counters in MongoDB

MongoDB does not have a built-in auto-increment like SQL databases, but you can build a reliable atomic counter using `findOneAndUpdate` with `$inc`. This pattern is useful for generating sequential IDs, tracking view counts, managing inventory quantities, and any scenario where multiple clients must safely increment a shared value.

## Basic Atomic Counter

Create a `counters` collection to store named counters:

```javascript
const { MongoClient } = require('mongodb');

async function getNextSequence(db, counterName) {
    const result = await db.collection('counters').findOneAndUpdate(
        { _id: counterName },
        { $inc: { seq: 1 } },
        { upsert: true, returnDocument: 'after' }
    );
    return result.seq;
}

// Usage
const client = new MongoClient(process.env.MONGO_URI);
await client.connect();
const db = client.db('mydb');

const nextOrderId = await getNextSequence(db, 'orders');
console.log('Next order ID:', nextOrderId); // 1, 2, 3, ...
```

## Inserting a Document with an Auto-Increment ID

```javascript
async function createOrder(db, orderData) {
    const seq = await getNextSequence(db, 'orders');
    const order = {
        orderId: seq,
        ...orderData,
        createdAt: new Date()
    };

    await db.collection('orders').insertOne(order);
    return order;
}
```

## Decrementing a Counter (Inventory Management)

Use `$inc` with a negative value and check the result to prevent going below zero:

```javascript
async function decrementInventory(db, productId, quantity) {
    const result = await db.collection('products').findOneAndUpdate(
        {
            _id: productId,
            stock: { $gte: quantity } // Only update if enough stock exists
        },
        { $inc: { stock: -quantity } },
        { returnDocument: 'after' }
    );

    if (!result) {
        throw new Error('Insufficient stock or product not found');
    }

    return result;
}
```

## Multi-Counter Document

Track multiple counters in a single document for related values:

```javascript
async function incrementPostStats(db, postId, field) {
    const validFields = ['views', 'likes', 'shares'];
    if (!validFields.includes(field)) {
        throw new Error('Invalid counter field');
    }

    const update = { $inc: { [field]: 1 } };
    const result = await db.collection('posts').findOneAndUpdate(
        { _id: postId },
        update,
        { returnDocument: 'after', upsert: false }
    );

    if (!result) {
        throw new Error('Post not found');
    }

    return result;
}
```

## Reset and Conditional Counter

Reset a counter or set it to a specific value atomically:

```javascript
async function resetCounter(db, counterName) {
    const result = await db.collection('counters').findOneAndUpdate(
        { _id: counterName },
        { $set: { seq: 0 } },
        { upsert: true, returnDocument: 'after' }
    );
    return result.seq;
}

// Set to specific value only if current value is lower
async function setCounterIfHigher(db, counterName, value) {
    const result = await db.collection('counters').findOneAndUpdate(
        { _id: counterName, seq: { $lt: value } },
        { $set: { seq: value } },
        { upsert: false, returnDocument: 'after' }
    );
    return result;
}
```

## Creating an Index for Performance

Ensure your counters collection has an index on `_id` (it does by default), and for product inventory, index the fields used in queries:

```javascript
// Index for fast inventory lookups with stock check
await db.collection('products').createIndex(
    { _id: 1, stock: 1 }
);
```

## Avoiding Counter Collisions in Sharded Clusters

In a sharded environment, use the counter document's shard key carefully. Place counter documents on a single shard or use a dedicated counters cluster, since cross-shard atomic operations are expensive:

```javascript
// For sharded setups, include the shard key in the filter
async function getNextShardedSequence(db, namespace, shardKey) {
    const result = await db.collection('counters').findOneAndUpdate(
        { _id: `${namespace}:${shardKey}` },
        { $inc: { seq: 1 } },
        { upsert: true, returnDocument: 'after' }
    );
    return result.seq;
}
```

## Summary

The `findOneAndUpdate` with `$inc` pattern provides a reliable, atomic counter implementation in MongoDB without requiring transactions. It handles concurrent requests safely because MongoDB document-level operations are atomic. For high-throughput scenarios, consider batching counter increments or using a dedicated Redis counter with periodic sync to MongoDB.
