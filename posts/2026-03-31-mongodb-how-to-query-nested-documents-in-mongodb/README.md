# How to Query Nested Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Nested Documents, Dot Notation, Embedded Documents, Query

Description: Learn how to query nested and embedded documents in MongoDB using dot notation, $elemMatch, and nested field filters for precise subdocument matching.

---

## Overview

MongoDB documents can contain embedded subdocuments at any level of nesting. Querying these fields uses dot notation to traverse the document hierarchy. Understanding how to query nested structures correctly - especially arrays of subdocuments - is essential for working with MongoDB's flexible document model.

## Sample Data Structure

```javascript
// Example user document with nested fields
{
  "_id": ObjectId("..."),
  "name": "Alice",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "zip": "10001",
    "geo": {
      "lat": 40.7128,
      "lon": -74.0060
    }
  },
  "orders": [
    { "orderId": "o1", "status": "shipped", "amount": 100 },
    { "orderId": "o2", "status": "pending", "amount": 50 }
  ],
  "tags": ["vip", "newsletter"]
}
```

## Querying with Dot Notation

Use dot notation to query fields inside embedded documents:

```javascript
// Query on a nested field
db.users.find({ "address.city": "New York" });

// Multiple levels of nesting
db.users.find({ "address.geo.lat": { $gte: 40 } });

// Combine nested and top-level filters
db.users.find({
  "address.city": "New York",
  name: { $regex: /^A/, $options: "i" }
});
```

## Exact Subdocument Match

Matching a full subdocument requires exact field order and values - use dot notation instead:

```javascript
// EXACT match: requires same field order and no extra fields
db.users.find({ address: { city: "New York", zip: "10001" } });
// Only matches if address has EXACTLY those two fields in that order

// BETTER: use dot notation for flexible matching
db.users.find({ "address.city": "New York", "address.zip": "10001" });
```

## Querying Arrays of Subdocuments

For arrays of embedded documents, dot notation matches if ANY element satisfies the condition:

```javascript
// Find users with at least one order with amount > 75
db.users.find({ "orders.amount": { $gt: 75 } });

// Find users with at least one shipped order
db.users.find({ "orders.status": "shipped" });
```

## Using $elemMatch for Multiple Conditions on the Same Element

Without `$elemMatch`, conditions apply across different array elements. Use `$elemMatch` to require all conditions on the same element:

```javascript
// Without $elemMatch: matches if ANY element has status "shipped" AND ANY element has amount > 75
// (could be different elements)
db.users.find({
  "orders.status": "shipped",
  "orders.amount": { $gt: 75 }
});

// With $elemMatch: the SAME array element must satisfy both conditions
db.users.find({
  orders: {
    $elemMatch: { status: "shipped", amount: { $gt: 75 } }
  }
});
```

## Querying Deeply Nested Arrays

```javascript
// Document structure with deeper nesting
{
  "company": "Acme",
  "departments": [
    {
      "name": "Engineering",
      "employees": [
        { "id": "e1", "level": "senior" },
        { "id": "e2", "level": "junior" }
      ]
    }
  ]
}

// Find companies with a senior engineer in Engineering
db.companies.find({
  departments: {
    $elemMatch: {
      name: "Engineering",
      "employees.level": "senior"
    }
  }
});
```

## Projecting Nested Fields

```javascript
// Return only nested fields
db.users.find(
  { "address.city": "New York" },
  { "address.city": 1, "address.zip": 1, name: 1, _id: 0 }
);

// Return a specific array element using $slice
db.users.find(
  {},
  { orders: { $slice: 1 } }  // return only the first order
);
```

## Aggregation with Nested Fields

```javascript
// Group by nested field
db.users.aggregate([
  { $group: { _id: "$address.city", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]);

// Unwind and filter array subdocuments
db.users.aggregate([
  { $unwind: "$orders" },
  { $match: { "orders.status": "shipped" } },
  { $group: { _id: "$_id", totalShipped: { $sum: "$orders.amount" } } }
]);
```

## Creating Indexes on Nested Fields

```javascript
// Index on a nested scalar field
db.users.createIndex({ "address.city": 1 });

// Compound index on nested fields
db.users.createIndex({ "address.city": 1, "address.zip": 1 });

// Index on a field inside an array of subdocuments (multikey index)
db.users.createIndex({ "orders.status": 1, "orders.amount": -1 });
```

## Summary

Querying nested documents in MongoDB uses dot notation to traverse subdocument fields at any depth. For scalar nested fields, dot notation works directly. For arrays of subdocuments, dot notation matches if any element satisfies the condition; use `$elemMatch` when multiple conditions must apply to the same array element. Always create indexes on nested fields that appear in frequent query filters to avoid collection scans.
