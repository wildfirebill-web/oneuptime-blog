# How to Use the find Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Find, Query, Projection

Description: A complete guide to the MongoDB find command - filtering documents, projections, sorting, limiting, and using query operators for precise data retrieval.

---

## Basic find Syntax

The `find()` method retrieves documents matching a filter. Without arguments, it returns all documents:

```javascript
// Return all documents
db.orders.find();

// Return documents matching a filter
db.orders.find({ status: "pending" });

// findOne returns a single document
db.orders.findOne({ status: "pending" });
```

## Comparison Operators

```javascript
// Exact match
db.products.find({ price: 29.99 });

// Greater than
db.products.find({ price: { $gt: 100 } });

// Range
db.products.find({ price: { $gte: 10, $lte: 50 } });

// Not equal
db.orders.find({ status: { $ne: "cancelled" } });

// In a list
db.orders.find({ status: { $in: ["pending", "processing"] } });

// Not in a list
db.orders.find({ status: { $nin: ["cancelled", "refunded"] } });
```

## Logical Operators

```javascript
// AND (implicit - multiple conditions in same document)
db.orders.find({ status: "active", total: { $gt: 100 } });

// Explicit AND
db.orders.find({ $and: [{ status: "active" }, { total: { $gt: 100 } }] });

// OR
db.orders.find({ $or: [{ status: "pending" }, { status: "processing" }] });

// NOR
db.orders.find({ $nor: [{ status: "cancelled" }, { status: "refunded" }] });
```

## Projections

The second argument controls which fields are returned. Use `1` to include and `0` to exclude:

```javascript
// Include only name and price (always includes _id by default)
db.products.find({}, { name: 1, price: 1 });

// Exclude _id
db.products.find({}, { name: 1, price: 1, _id: 0 });

// Exclude specific fields
db.users.find({}, { password: 0, secretToken: 0 });
```

## Sorting, Limiting, and Skipping

```javascript
// Sort ascending by price
db.products.find().sort({ price: 1 });

// Sort descending by date
db.orders.find().sort({ createdAt: -1 });

// Limit results
db.orders.find().limit(10);

// Pagination
const page = 2;
const pageSize = 20;
db.orders.find().skip((page - 1) * pageSize).limit(pageSize);
```

## Querying Nested Documents

```javascript
// Dot notation for nested fields
db.users.find({ "address.city": "New York" });
db.orders.find({ "shipping.status": "delivered" });
```

## Querying Arrays

```javascript
// Array contains a value
db.products.find({ tags: "electronics" });

// All elements present
db.products.find({ tags: { $all: ["electronics", "sale"] } });

// Array element by index
db.products.find({ "images.0.width": 800 });

// Array size
db.orders.find({ items: { $size: 3 } });
```

## Counting Results

```javascript
// Count all matching documents
db.orders.countDocuments({ status: "pending" });

// Estimated count (faster, uses metadata)
db.orders.estimatedDocumentCount();
```

## Using Cursors

```javascript
// forEach with cursor
db.orders.find({ status: "pending" }).forEach(doc => {
  print(doc._id + ": " + doc.total);
});

// Convert to array
const results = db.orders.find({ status: "pending" }).toArray();
```

## Summary

MongoDB's `find()` command supports rich query expressions using comparison operators (`$gt`, `$lte`, `$in`), logical operators (`$and`, `$or`), and array operators (`$all`, `$size`). Use projections to limit returned fields, chain `.sort()`, `.limit()`, and `.skip()` for pagination, and access nested fields with dot notation.
