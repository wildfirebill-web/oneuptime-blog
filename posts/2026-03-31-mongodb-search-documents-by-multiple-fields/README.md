# How to Search for Documents by Multiple Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Index, Compound, Operator

Description: Learn how to query MongoDB documents using multiple field conditions with AND logic, OR logic, and compound indexes for optimal performance.

---

Searching documents by multiple fields is the backbone of most MongoDB applications. Whether you need all conditions to match (AND logic) or any condition to match (OR logic), MongoDB provides a clean, expressive query syntax.

## AND Queries (Multiple Field Conditions)

By default, conditions in a query document are ANDed together:

```javascript
// Find active users in the "admin" role
db.users.find({
  status: "active",
  role: "admin"
})

// With numeric conditions
db.products.find({
  category: "electronics",
  price: { $lte: 200 },
  inStock: true
})
```

This is equivalent to SQL: `WHERE status = 'active' AND role = 'admin'`.

## Explicit $and Operator

Use `$and` when you need multiple conditions on the same field, or for clarity:

```javascript
// Multiple conditions on the same field require $and
db.products.find({
  $and: [
    { price: { $gte: 50 } },
    { price: { $lte: 200 } }
  ]
})

// Equivalent shorthand (works because different operators)
db.products.find({
  price: { $gte: 50, $lte: 200 }
})
```

## OR Queries

Use `$or` to match documents satisfying any of the listed conditions:

```javascript
// Find users who are admins OR have premium status
db.users.find({
  $or: [
    { role: "admin" },
    { subscription: "premium" }
  ]
})
```

## Combining AND with OR

Combining is straightforward - mix conditions in the query object:

```javascript
// Active users who are (admins OR premium subscribers)
db.users.find({
  status: "active",
  $or: [
    { role: "admin" },
    { subscription: "premium" }
  ]
})
```

## Searching Across Multiple Fields with $or

A common search box pattern - match a query string across multiple fields:

```javascript
function multiFieldSearch(searchTerm) {
  const regex = new RegExp(searchTerm, "i");
  return db.collection("products").find({
    $or: [
      { name: regex },
      { description: regex },
      { sku: regex },
      { brand: regex }
    ]
  });
}

const results = await multiFieldSearch("samsung").toArray();
```

## Compound Indexes for Multi-Field Queries

For frequent AND queries, a compound index dramatically improves performance:

```javascript
// Compound index for the query: { status: "active", role: "admin" }
db.users.createIndex({ status: 1, role: 1 })

// Multi-field query with range condition
db.orders.createIndex({ customerId: 1, placedAt: -1 })

// Query uses both parts of the compound index
db.orders.find({
  customerId: "cust-123",
  placedAt: { $gte: new Date("2024-01-01") }
}).sort({ placedAt: -1 })
```

## Index Strategy for $or Queries

For `$or` queries, each condition should have its own index:

```javascript
db.users.createIndex({ role: 1 })
db.users.createIndex({ subscription: 1 })

// MongoDB can use a separate index for each $or clause
db.users.find({
  $or: [
    { role: "admin" },
    { subscription: "premium" }
  ]
}).explain("executionStats")
```

## Using the Aggregation Pipeline for Complex Multi-Field Search

```javascript
db.products.aggregate([
  {
    $match: {
      category: "electronics",
      brand: { $in: ["Samsung", "Apple", "Sony"] },
      price: { $gte: 100, $lte: 1000 },
      "availability.inStock": true
    }
  },
  { $sort: { price: 1 } },
  { $limit: 20 }
])
```

## Summary

MongoDB ANDs multiple field conditions by default in a query document. Use `$or` when any condition should match. For frequent multi-field AND queries, create compound indexes and order fields with the most selective first. For `$or` queries, ensure each branch has its own index. Use `explain()` to verify which indexes are selected and that no unexpected collection scans occur.
