# How to Use $nin to Exclude Values in MongoDB Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $nin, NOT IN, Query Operator, Exclusion Queries, Node.js

Description: Learn how to use the MongoDB $nin operator to exclude documents matching any value in a specified list, with examples, array behavior, and performance considerations.

---

## What Is the $nin Operator?

The `$nin` (not in) operator selects documents where the field value **does not equal any value** in the specified array. It also matches documents where the field does not exist. It is the inverse of `$in`.

```javascript
// Exclude cancelled and refunded orders
db.orders.find({ status: { $nin: ['cancelled', 'refunded'] } })
```

## Basic Usage

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('shop');
const products = db.collection('products');

// Exclude specific categories
const nonElectronics = await products.find({
  category: { $nin: ['electronics', 'computers'] }
}).toArray();

// Exclude specific numeric values
const notOnSalePrices = await products.find({
  price: { $nin: [9.99, 19.99] }
}).toArray();

// Exclude documents by specific IDs
const { ObjectId } = require('mongodb');
const excludedIds = ['64f1a2b3c4d5e6f7a8b9c0d1', '64f1a2b3c4d5e6f7a8b9c0d2']
  .map(id => new ObjectId(id));

const withoutExcluded = await products.find({
  _id: { $nin: excludedIds }
}).toArray();
```

## $nin with Array Fields

When the document field is an array, `$nin` matches documents where NONE of the specified values appear in the array:

```javascript
const docs = [
  { _id: 1, tags: ['sale', 'new'] },
  { _id: 2, tags: ['clearance'] },
  { _id: 3, tags: ['featured', 'premium'] },
];

// Returns doc 3 only - no 'sale' or 'clearance' in tags
await products.find({ tags: { $nin: ['sale', 'clearance'] } }).toArray();
```

## Handling Null and Missing Fields

`$nin` also matches documents where the field is absent or null:

```javascript
// This matches documents where:
// - status is not 'cancelled'
// - status is not 'refunded'
// - status field does not exist
// - status is null
const results = await db.collection('orders').find({
  status: { $nin: ['cancelled', 'refunded'] }
}).toArray();

// To explicitly exclude null/missing too:
const results2 = await db.collection('orders').find({
  status: { $nin: ['cancelled', 'refunded', null] }
}).toArray();
```

## Using $nin in Aggregation

```javascript
// Filter with $match
const results = await products.aggregate([
  { $match: { status: { $nin: ['discontinued', 'out_of_stock'] } } },
  { $group: { _id: '$category', count: { $sum: 1 } } },
]).toArray();

// Boolean expression in $project
const result = await products.aggregate([
  {
    $project: {
      name: 1,
      isAvailable: {
        $not: [{ $in: ['$status', ['discontinued', 'out_of_stock']] }]
      },
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

# Exclude certain categories
results = list(products.find({'category': {'$nin': ['electronics', 'computers']}}))

# Exclude specific statuses
active_orders = list(db['orders'].find({
    'status': {'$nin': ['cancelled', 'refunded', 'failed']}
}))
```

## Practical Example - Product Feed Excluding Known Bad Items

```javascript
async function getProductFeed(excludedProductIds, excludedCategories) {
  const { ObjectId } = require('mongodb');

  return db.collection('products').find({
    _id: { $nin: excludedProductIds.map(id => new ObjectId(id)) },
    category: { $nin: excludedCategories },
    status: { $nin: ['discontinued', 'draft', 'out_of_stock'] },
  }).sort({ createdAt: -1 }).limit(50).toArray();
}
```

## Index Behavior

Like `$ne`, `$nin` queries are generally not as index-efficient as `$in` because they typically match most of the collection:

```javascript
const plan = await products
  .find({ category: { $nin: ['electronics'] } })
  .explain('executionStats');

// Often results in COLLSCAN (full collection scan)
```

When only a small fraction of the collection matches `$nin`, MongoDB may still use an index. For large exclusion lists with many matching documents, a collection scan is often chosen by the query planner.

**Performance tip:** If you know the set of values you DO want, use `$in` instead:

```javascript
// Instead of excluding 1 category from 10:
{ category: { $nin: ['electronics'] } }

// Prefer listing the 9 categories you want:
{ category: { $in: ['clothing', 'books', 'toys', ...] } }
```

## Summary

The MongoDB `$nin` operator excludes documents where a field matches any value in a specified list, and also matches documents where the field is missing or null. It is the inverse of `$in` and is most useful when you have a small exclusion list. Be aware that `$nin` queries often result in full collection scans; when the set of desired values is known, using `$in` is typically more performant.
