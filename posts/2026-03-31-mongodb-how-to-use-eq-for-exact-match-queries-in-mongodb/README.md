# How to Use $eq for Exact Match Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $eq, Query Operator, Exact Match, Filter, Node.js

Description: Learn how to use the MongoDB $eq operator for exact field matching, including its implicit usage, explicit syntax, and differences from equality shorthand.

---

## What Is the $eq Operator?

The `$eq` operator matches documents where the value of a field equals the specified value. It is the explicit form of the simple equality match and accepts any BSON type.

```javascript
// Implicit equality (shorthand) - most common
db.collection.find({ status: 'active' })

// Explicit $eq - equivalent result
db.collection.find({ status: { $eq: 'active' } })
```

Both forms produce identical query plans. The shorthand is preferred for simple cases; explicit `$eq` is useful inside nested expressions.

## Basic Usage

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('shop');
const products = db.collection('products');

// Match on a string field
const electronics = await products
  .find({ category: { $eq: 'electronics' } })
  .toArray();

// Match on a number
const freeItems = await products
  .find({ price: { $eq: 0 } })
  .toArray();

// Match on a boolean
const activeItems = await products
  .find({ inStock: { $eq: true } })
  .toArray();

// Match on ObjectId
const { ObjectId } = require('mongodb');
const item = await products
  .findOne({ _id: { $eq: new ObjectId('64f1a2b3c4d5e6f7a8b9c0d1') } });
```

## Matching Nested Fields

Use dot notation to match nested document fields:

```javascript
// Match nested field
const usa = await products
  .find({ 'address.country': { $eq: 'US' } })
  .toArray();

// Match array element exactly
const tagged = await products
  .find({ tags: { $eq: 'sale' } })
  .toArray();
// This matches documents where 'sale' is ANY element in the tags array
```

## $eq in Aggregation Pipelines

In aggregation, `$eq` is used as an expression operator returning a boolean:

```javascript
const results = await products.aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      isFree: { $eq: ['$price', 0] },      // returns true/false
      isPremium: { $eq: ['$tier', 'premium'] },
    }
  }
]).toArray();
// Each document now has isFree: true/false field
```

### Filtering with $eq in $match

```javascript
const results = await products.aggregate([
  { $match: { status: { $eq: 'active' } } },
  { $group: { _id: '$category', total: { $sum: 1 } } },
]).toArray();
```

### Using $eq in $cond (conditional)

```javascript
const results = await products.aggregate([
  {
    $project: {
      name: 1,
      label: {
        $cond: {
          if: { $eq: ['$status', 'active'] },
          then: 'Available',
          else: 'Unavailable',
        }
      }
    }
  }
]).toArray();
```

## $eq with Arrays

When the field is an array, `$eq` matches documents where the specified value appears anywhere in the array:

```javascript
const docs = [
  { _id: 1, tags: ['sale', 'new'] },
  { _id: 2, tags: ['clearance'] },
  { _id: 3, tags: ['sale'] },
];

// Matches documents 1 and 3
await products.find({ tags: { $eq: 'sale' } }).toArray();
```

To match an exact array (same elements in same order), use:

```javascript
// Exact array match - must have exactly ['sale', 'new'] in that order
await products.find({ tags: { $eq: ['sale', 'new'] } }).toArray();
```

## PyMongo Usage

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017')
db = client['shop']
products = db['products']

# Simple equality match
results = list(products.find({'status': {'$eq': 'active'}}))

# Nested field
results = list(products.find({'address.country': {'$eq': 'US'}}))
```

## Comparison: $eq vs. Shorthand

| Feature | Shorthand `{field: value}` | Explicit `{field: {$eq: value}}` |
|---------|---------------------------|----------------------------------|
| Readability | Simpler for basic cases | More verbose |
| Index usage | Yes | Yes (identical) |
| In expressions | Cannot be used | Can be used inside $cond, $addFields, etc. |
| Performance | Identical | Identical |

## Summary

The MongoDB `$eq` operator matches documents where a field equals an exact value. For simple queries, the implicit shorthand `{ field: value }` is equivalent and preferred for readability. Use the explicit `{ field: { $eq: value } }` form when nesting inside aggregation expressions like `$cond`, `$addFields`, or `$project`. Both forms use indexes identically and produce the same query execution plans.
