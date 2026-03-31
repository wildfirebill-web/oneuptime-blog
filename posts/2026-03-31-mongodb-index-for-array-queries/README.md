# How to Index for Array Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Index, Multikey, Query

Description: Understand how MongoDB multikey indexes work for array fields, their compound index limitations, and how to index nested array documents with $elemMatch.

---

## Multikey Indexes

When you create an index on a field that contains an array, MongoDB automatically creates a multikey index. It stores one index entry per element in the array, enabling queries on individual array elements.

```javascript
db.products.insertMany([
  { name: "Widget", tags: ["electronics", "sale", "new"] },
  { name: "Gadget", tags: ["electronics", "premium"] },
  { name: "Doohickey", tags: ["sale", "clearance"] },
]);

// This creates a multikey index automatically
db.products.createIndex({ tags: 1 });
```

MongoDB internally stores three index entries for the Widget: one for "electronics", one for "sale", one for "new".

## Querying with Multikey Index

```javascript
// Find products in the "sale" tag - uses the multikey index efficiently
db.products.find({ tags: "sale" }).explain("executionStats");
// Stage: IXSCAN, totalDocsExamined ≈ nReturned

// $all requires ALL specified values to be present
db.products.find({ tags: { $all: ["electronics", "sale"] } });

// $in matches if ANY specified value is present
db.products.find({ tags: { $in: ["electronics", "premium"] } });
```

## Compound Multikey Index Limitation

A compound index cannot have more than one field that is an array across any single document. This is a server-level restriction:

```javascript
// Index created successfully
db.products.createIndex({ tags: 1, sizes: 1 });

// Insert FAILS if both tags and sizes are arrays in the same document
db.products.insertOne({
  name: "Widget",
  tags: ["electronics"],  // array
  sizes: ["S", "M", "L"], // array - INVALID with compound multikey index
});
// Error: cannot index parallel arrays [tags] [sizes]
```

Solution: embed one array field's elements as a scalar if possible, or restructure the schema.

## Indexing Arrays of Embedded Documents

For arrays of objects, create an index on a field within the array using dot notation:

```javascript
db.orders.insertOne({
  customer: "Alice",
  items: [
    { productId: "p1", qty: 2 },
    { productId: "p2", qty: 5 },
  ],
});

// Index on a field within the array
db.orders.createIndex({ "items.productId": 1 });

// Efficient query using the index
db.orders.find({ "items.productId": "p1" });
```

## Using $elemMatch for Multi-Condition Array Queries

When you need to match multiple conditions on the same array element, use `$elemMatch`:

```javascript
// Find orders where the same item has productId "p1" AND qty > 3
db.orders.find({
  items: { $elemMatch: { productId: "p1", qty: { $gt: 3 } } },
});
```

Without `$elemMatch`, MongoDB matches documents where one element satisfies the first condition and any element satisfies the second:

```javascript
// This matches if items contains productId "p1" in ONE element and qty > 3 in ANY element
db.orders.find({ "items.productId": "p1", "items.qty": { $gt: 3 } }); // Incorrect!
```

## Compound Index with $elemMatch

Create a compound index on multiple fields within an array to support `$elemMatch`:

```javascript
db.orders.createIndex({ "items.productId": 1, "items.qty": 1 });

// The index may be used for this $elemMatch query
db.orders.find({
  items: { $elemMatch: { productId: "p1", qty: { $gt: 3 } } },
}).explain("executionStats");
```

## Array Size Queries

`$size` does not use an index and requires a collection scan. For size-based filtering, track the array length as a separate field:

```javascript
// Store array length explicitly
db.products.updateMany({}, [
  { $set: { tagCount: { $size: "$tags" } } },
]);

db.products.createIndex({ tagCount: 1 });
db.products.find({ tagCount: { $gte: 3 } });
```

## Summary

MongoDB multikey indexes index individual array elements, enabling efficient element-level queries. A compound index cannot span two array fields in the same document. For arrays of embedded documents, index nested fields using dot notation and use `$elemMatch` when multiple conditions must match the same array element. Track array length as a separate field for efficient size-based range queries, since `$size` cannot use an index.
