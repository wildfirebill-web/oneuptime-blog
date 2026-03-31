# How to Use $and to Combine Multiple Conditions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $and, Logical Operator, Query, Multiple Conditions, Node.js

Description: Learn how to use MongoDB's $and operator to combine multiple query conditions, when it is necessary versus the implicit AND, and how it interacts with indexes.

---

## What Is the $and Operator?

The `$and` operator joins multiple query conditions and returns documents that satisfy ALL conditions. MongoDB applies an implicit AND when you specify multiple fields in a single query document - explicit `$and` is needed only in specific cases.

```javascript
// Implicit AND (most common - fields are joined with AND automatically)
db.products.find({ status: 'active', price: { $lt: 100 } })

// Explicit $and (equivalent result)
db.products.find({
  $and: [
    { status: 'active' },
    { price: { $lt: 100 } },
  ]
})
```

## When Explicit $and Is Required

Explicit `$and` is required when you need to apply multiple conditions **on the same field**. MongoDB's query document model does not allow duplicate keys, so you cannot write:

```javascript
// WRONG - duplicate key, second condition overwrites first
db.products.find({ price: { $gt: 10 }, price: { $lt: 100 } })

// CORRECT - use $and for multiple conditions on the same field
db.products.find({
  $and: [
    { price: { $gt: 10 } },
    { price: { $lt: 100 } },
  ]
})

// ALSO CORRECT - MongoDB allows range operators in same document
db.products.find({ price: { $gt: 10, $lt: 100 } })
```

The cleaner form for range queries is to combine operators in a single object. Use explicit `$and` when combining different operators that cannot be merged:

```javascript
// Multiple $elemMatch conditions on same field require $and
db.products.find({
  $and: [
    { reviews: { $elemMatch: { rating: { $gte: 4 } } } },
    { reviews: { $elemMatch: { verified: true } } },
  ]
})
```

## Practical Examples

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('shop');
const orders = db.collection('orders');

// Active orders over $100 (implicit AND)
const bigActiveOrders = await orders.find({
  status: 'active',
  total: { $gt: 100 },
}).toArray();

// Same with explicit $and
const explicit = await orders.find({
  $and: [
    { status: 'active' },
    { total: { $gt: 100 } },
    { createdAt: { $gte: new Date('2024-01-01') } },
  ]
}).toArray();

// Combining $and with $or
const complex = await orders.find({
  $and: [
    { total: { $gt: 50 } },
    {
      $or: [
        { status: 'processing' },
        { status: 'shipped' },
      ]
    }
  ]
}).toArray();
```

## Using $and in Aggregation

```javascript
// $match with $and
const results = await db.collection('products').aggregate([
  {
    $match: {
      $and: [
        { price: { $gte: 10, $lte: 200 } },
        { status: 'active' },
        { category: { $in: ['electronics', 'gadgets'] } },
      ]
    }
  },
  { $sort: { price: 1 } },
]).toArray();
```

In expression context:

```javascript
const results = await db.collection('products').aggregate([
  {
    $project: {
      name: 1,
      isBestDeal: {
        $and: [
          { $lte: ['$price', 50] },
          { $eq: ['$status', 'active'] },
          { $gte: ['$rating', 4] },
        ]
      }
    }
  }
]).toArray();
```

## PyMongo Usage

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017')
db = client['shop']
products = db['products']

# Implicit AND
results = list(products.find({
    'status': 'active',
    'price': {'$lt': 100}
}))

# Explicit $and for same-field conditions
results = list(products.find({
    '$and': [
        {'tags': {'$elemMatch': {'$eq': 'sale'}}},
        {'tags': {'$elemMatch': {'$eq': 'featured'}}},
    ]
}))
```

## Index Behavior

MongoDB's query optimizer handles `$and` conditions well:

```javascript
// Create a compound index for this common query
await db.collection('orders').createIndex({ status: 1, total: 1 });

// This query will use the compound index
await orders.find({ status: 'active', total: { $gt: 100 } })
  .explain('executionStats');
// Stage: IXSCAN using the compound index
```

For `$and` with conditions on different fields, MongoDB may use multiple single-field indexes and merge the results, but a compound index is more efficient.

## Common Mistakes

```javascript
// MISTAKE: Duplicate keys - second overwrites first silently
const wrong = await orders.find({ status: 'active', status: 'pending' });
// This only filters status: 'pending' !

// CORRECT: Use $or or $in for multiple values on same field
const correct = await orders.find({ status: { $in: ['active', 'pending'] } });
```

## Summary

MongoDB applies an implicit AND when you specify multiple fields in a single query document, making explicit `$and` unnecessary for most queries. Use explicit `$and` when you need to apply multiple conditions on the same field, such as combining two `$elemMatch` expressions, or when combining `$or` with other conditions for clarity. Compound indexes on frequently combined fields significantly improve `$and` query performance.
