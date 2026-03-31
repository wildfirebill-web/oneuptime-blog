# How to Use $expr to Compare Two Fields in the Same Document in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $expr, Field Comparison, Aggregation Expressions, Query, Node.js

Description: Learn how to use MongoDB's $expr operator to compare two fields within the same document using aggregation expressions in find queries and aggregation pipelines.

---

## What Is the $expr Operator?

The `$expr` operator allows you to use aggregation expressions within the query language. Its most common use is comparing two fields in the same document - something that regular query operators cannot do.

```javascript
// Find orders where discount > total (a data anomaly)
db.orders.find({ $expr: { $gt: ['$discount', '$total'] } })
```

Without `$expr`, you cannot write `{ discount: { $gt: '$total' } }` - MongoDB treats the `$total` string as a literal value.

## Basic Field-to-Field Comparisons

```javascript
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');
const db = client.db('shop');
const orders = db.collection('orders');

// Find orders where the discount exceeds the subtotal (bad data)
const overDiscounted = await orders.find({
  $expr: { $gt: ['$discount', '$subtotal'] }
}).toArray();

// Find employees who earn more than their manager
const highEarners = await db.collection('employees').find({
  $expr: { $gt: ['$salary', '$manager.salary'] }
}).toArray();

// Find products where quantity sold exceeds quantity in stock
const oversold = await db.collection('products').find({
  $expr: { $gt: ['$soldCount', '$stockCount'] }
}).toArray();

// Find tasks where actual duration exceeded estimated duration
const overTime = await db.collection('tasks').find({
  $expr: { $gt: ['$actualHours', '$estimatedHours'] }
}).toArray();
```

## Using $expr with Date Fields

```javascript
// Find subscriptions that have already expired (endDate < today)
const expired = await db.collection('subscriptions').find({
  $expr: { $lt: ['$endDate', new Date()] }
}).toArray();

// Find orders where ship date is before order date (invalid data)
const badDates = await orders.find({
  $expr: { $lt: ['$shippedAt', '$createdAt'] }
}).toArray();
```

## Complex Expressions with $expr

Since `$expr` accepts any aggregation expression, you can perform calculations before comparing:

```javascript
// Find products where discounted price < cost (selling at a loss)
const atLoss = await db.collection('products').find({
  $expr: {
    $lt: [
      { $multiply: ['$price', { $subtract: [1, '$discountRate'] }] }, // discounted price
      '$costPrice'  // cost
    ]
  }
}).toArray();

// Find users who have spent more than their credit limit
const overLimit = await db.collection('accounts').find({
  $expr: {
    $gt: [
      { $add: ['$balance', '$pendingCharges'] },  // effective balance
      '$creditLimit'
    ]
  }
}).toArray();

// Find orders where tax is more than 20% of subtotal (suspicious)
const highTax = await orders.find({
  $expr: {
    $gt: [
      '$tax',
      { $multiply: ['$subtotal', 0.2] }
    ]
  }
}).toArray();
```

## $expr in Aggregation Pipelines

Within aggregation, `$expr` is used in `$match` stages:

```javascript
const results = await db.collection('projects').aggregate([
  {
    $match: {
      $expr: {
        $gt: ['$actualCost', '$budget']   // over-budget projects
      }
    }
  },
  {
    $project: {
      name: 1,
      budget: 1,
      actualCost: 1,
      overBudgetBy: { $subtract: ['$actualCost', '$budget'] },
    }
  },
  { $sort: { overBudgetBy: -1 } },
]).toArray();
```

## String Field Comparisons

```javascript
// Find documents where firstName = lastName
const sameNames = await db.collection('users').find({
  $expr: { $eq: ['$firstName', '$lastName'] }
}).toArray();

// Find inventory where reorderPoint >= currentStock (needs reorder)
const reorderNeeded = await db.collection('inventory').find({
  $expr: { $gte: ['$reorderPoint', '$currentStock'] }
}).toArray();
```

## Combining $expr with Other Query Operators

```javascript
// Active subscriptions that are about to expire (within 7 days)
const now = new Date();
const sevenDaysFromNow = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000);

const expiringSoon = await db.collection('subscriptions').find({
  status: 'active',
  $expr: {
    $and: [
      { $gt: ['$endDate', now] },
      { $lte: ['$endDate', sevenDaysFromNow] },
    ]
  }
}).toArray();
```

## PyMongo Usage

```python
from pymongo import MongoClient
from datetime import datetime, timedelta

client = MongoClient('mongodb://localhost:27017')
db = client['shop']
orders = db['orders']

# Compare two fields
over_discounted = list(orders.find({
    '$expr': {'$gt': ['$discount', '$subtotal']}
}))

# Calculate and compare
at_loss = list(db['products'].find({
    '$expr': {
        '$lt': [
            {'$multiply': ['$price', {'$subtract': [1, '$discountRate']}]},
            '$costPrice'
        ]
    }
}))

# Date comparison
expired = list(db['subscriptions'].find({
    '$expr': {'$lt': ['$endDate', datetime.utcnow()]}
}))
```

## Index Usage with $expr

`$expr` queries can use indexes in some cases, particularly with simple field comparisons. More complex expressions with arithmetic may not use indexes:

```javascript
// May use index if fields are indexed
await orders.createIndex({ subtotal: 1, discount: 1 });

const plan = await orders
  .find({ $expr: { $gt: ['$discount', '$subtotal'] } })
  .explain('executionStats');
// Inspect winning plan for IXSCAN vs COLLSCAN
```

## Summary

MongoDB's `$expr` operator enables field-to-field comparisons within the same document using the full power of aggregation expressions. It is the only way to compare two document fields in a query. Use it to detect data anomalies (discounts exceeding totals), enforce business rules (expiry date after creation date), and compute derived comparisons (discounted price vs cost). Be aware that complex arithmetic expressions in `$expr` may prevent index usage and result in collection scans on large collections.
