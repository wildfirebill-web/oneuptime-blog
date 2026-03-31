# How to Combine $and and $or for Complex Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $and, $or, Complex Queries, Logical Operators, Query, Node.js

Description: Learn how to combine MongoDB's $and and $or operators to build complex multi-condition queries, with real-world examples and query optimization patterns.

---

## Why Combine $and and $or?

Real-world queries often need to express logic like:

- "Find orders that are (pending OR processing) AND (total > 50 OR priority is urgent)"
- "Find users who are (admin OR manager) AND (active AND last seen within 30 days)"

MongoDB's logical operators can be nested to express any Boolean logic.

## AND of ORs

Match documents where multiple groups of alternatives are all satisfied:

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('shop');
const orders = db.collection('orders');

// (status is 'pending' OR 'processing') AND (total > 50 OR priority is 'urgent')
const important = await orders.find({
  $and: [
    {
      $or: [
        { status: 'pending' },
        { status: 'processing' },
      ]
    },
    {
      $or: [
        { total: { $gt: 50 } },
        { priority: 'urgent' },
      ]
    }
  ]
}).toArray();
```

## OR of ANDs

Match documents where at least one group of conditions is fully satisfied:

```javascript
// (category is 'electronics' AND price < 100)
// OR (category is 'books' AND rating >= 4)
const deals = await db.collection('products').find({
  $or: [
    {
      $and: [
        { category: 'electronics' },
        { price: { $lt: 100 } },
      ]
    },
    {
      $and: [
        { category: 'books' },
        { rating: { $gte: 4 } },
      ]
    }
  ]
}).toArray();
```

## Mixing Implicit AND with $or

The most common pattern: implicit AND with an explicit $or clause:

```javascript
// status MUST be 'active' (implicit AND)
// AND (price < 20 OR onSale is true)
const result = await db.collection('products').find({
  status: 'active',
  $or: [
    { price: { $lt: 20 } },
    { onSale: true },
  ]
}).toArray();
```

## Real-World Example: User Search

```javascript
async function searchUsers(searchTerm, filters = {}) {
  const conditions = [];

  // Text search condition
  if (searchTerm) {
    conditions.push({
      $or: [
        { name: { $regex: searchTerm, $options: 'i' } },
        { email: { $regex: searchTerm, $options: 'i' } },
        { username: { $regex: searchTerm, $options: 'i' } },
      ]
    });
  }

  // Role filter
  if (filters.roles && filters.roles.length > 0) {
    conditions.push({ role: { $in: filters.roles } });
  }

  // Activity filter
  if (filters.activeOnly) {
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    conditions.push({
      $or: [
        { lastLoginAt: { $gte: thirtyDaysAgo } },
        { status: 'admin' },  // admins always shown
      ]
    });
  }

  const query = conditions.length > 0
    ? { $and: conditions }
    : {};

  return db.collection('users')
    .find(query)
    .sort({ name: 1 })
    .limit(50)
    .toArray();
}
```

## E-Commerce Product Filter

```javascript
async function filterProducts({ categories, minPrice, maxPrice, tags, minRating }) {
  const must = [];

  // Category filter
  if (categories && categories.length > 0) {
    must.push({ category: { $in: categories } });
  }

  // Price range
  if (minPrice !== undefined || maxPrice !== undefined) {
    const priceFilter = {};
    if (minPrice !== undefined) priceFilter.$gte = minPrice;
    if (maxPrice !== undefined) priceFilter.$lte = maxPrice;
    must.push({ price: priceFilter });
  }

  // Tags - must match at least one
  if (tags && tags.length > 0) {
    must.push({ tags: { $in: tags } });
  }

  // Rating
  if (minRating !== undefined) {
    must.push({ rating: { $gte: minRating } });
  }

  // Always require active and in-stock
  must.push({ status: 'active' });
  must.push({ stock: { $gt: 0 } });

  const query = must.length > 0 ? { $and: must } : {};
  return db.collection('products').find(query).sort({ rating: -1 }).toArray();
}
```

## PyMongo Usage

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017')
db = client['shop']

# (status is 'pending' or 'processing') AND (total > 50 or priority is 'urgent')
orders = list(db['orders'].find({
    '$and': [
        {
            '$or': [
                {'status': 'pending'},
                {'status': 'processing'},
            ]
        },
        {
            '$or': [
                {'total': {'$gt': 50}},
                {'priority': 'urgent'},
            ]
        }
    ]
}))
```

## Aggregation Pipeline with Combined Logic

```javascript
const results = await db.collection('products').aggregate([
  {
    $match: {
      $and: [
        { status: 'active' },
        {
          $or: [
            { category: 'electronics', price: { $lt: 100 } },
            { category: 'books', rating: { $gte: 4 } },
          ]
        }
      ]
    }
  },
  { $sort: { rating: -1 } },
  { $limit: 20 },
]).toArray();
```

## Index Optimization Tips

```javascript
// For complex queries, create compound indexes matching your common access patterns
await db.collection('orders').createIndex({ status: 1, total: 1 });
await db.collection('products').createIndex({ status: 1, category: 1, price: 1 });
await db.collection('products').createIndex({ status: 1, rating: -1 });

// Use explain() to verify index usage
const plan = await db.collection('products')
  .find({ status: 'active', $or: [{ price: { $lt: 20 } }, { onSale: true }] })
  .explain('queryPlanner');

console.log(JSON.stringify(plan.queryPlanner.winningPlan, null, 2));
```

## Summary

Combining `$and` and `$or` in MongoDB allows you to express any Boolean query logic. The most common pattern is mixing implicit AND (top-level fields) with an explicit `$or` clause. For more complex logic, nest `$or` inside `$and` (AND of ORs) or `$and` inside `$or` (OR of ANDs). Build queries programmatically by constructing the conditions array dynamically to keep code clean. Use `explain()` to verify that compound indexes are being used for your combined conditions.
