# How to Store and Query Array Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Query, Multikey Index, Aggregation

Description: Learn how to store, query, and index array fields in MongoDB including element matching, array size filters, and multikey index behavior.

---

## Arrays in MongoDB

MongoDB natively supports array fields, allowing documents to embed ordered lists of any BSON type within a single field. This eliminates the need for many-to-many join tables common in relational databases.

## Inserting Array Values

```javascript
db.articles.insertMany([
  { title: "Intro to MongoDB", tags: ["database", "nosql", "mongodb"] },
  { title: "Advanced Indexing", tags: ["mongodb", "performance", "index"] },
  { title: "Aggregation Guide", tags: ["mongodb", "aggregation"] },
]);
```

## Querying Array Elements

```javascript
// Find documents where the array contains a specific value
db.articles.find({ tags: "mongodb" });

// Find documents where the array contains ALL specified values
db.articles.find({ tags: { $all: ["mongodb", "performance"] } });

// Find documents where the array contains AT LEAST ONE of the values
db.articles.find({ tags: { $in: ["nosql", "aggregation"] } });
```

## Querying by Array Size

```javascript
// Find articles with exactly 3 tags
db.articles.find({ tags: { $size: 3 } });

// Find articles with at least 2 tags (size is not directly rangeable)
db.articles.find({ "tags.1": { $exists: true } }); // At least 2 elements
```

## Querying Nested Array Elements

For arrays of embedded documents, use dot notation and `$elemMatch`:

```javascript
db.orders.insertOne({
  customer: "Alice",
  items: [
    { product: "Widget", qty: 5, price: 9.99 },
    { product: "Gadget", qty: 2, price: 24.99 },
  ],
});

// Find orders containing a Widget item with qty > 3
db.orders.find({
  items: { $elemMatch: { product: "Widget", qty: { $gt: 3 } } },
});
```

## Array Update Operators

```javascript
// Append a single element
db.articles.updateOne({ title: "Intro to MongoDB" }, { $push: { tags: "tutorial" } });

// Append multiple elements without duplicates
db.articles.updateOne(
  { title: "Intro to MongoDB" },
  { $addToSet: { tags: { $each: ["beginner", "nosql"] } } }
);

// Remove an element by value
db.articles.updateOne({ title: "Intro to MongoDB" }, { $pull: { tags: "tutorial" } });

// Remove the first or last element
db.articles.updateOne({ title: "Intro to MongoDB" }, { $pop: { tags: -1 } }); // -1 removes first, 1 removes last
```

## Multikey Indexes

When you create an index on an array field, MongoDB creates a multikey index with one entry per array element:

```javascript
db.articles.createIndex({ tags: 1 });
```

This enables efficient queries on any element:

```javascript
db.articles.find({ tags: "performance" }).explain("executionStats");
// winningPlan.stage should be IXSCAN
```

Limitation: compound multikey indexes cannot have more than one field that is an array.

## Aggregation with Arrays

```javascript
// Unwind the array into separate documents
db.articles.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]);
```

## Summary

MongoDB array fields store ordered lists within documents and support rich query operators including `$all`, `$in`, `$size`, and `$elemMatch`. Multikey indexes create one index entry per array element enabling efficient element-level queries. Use `$push`, `$pull`, `$addToSet`, and `$pop` for atomic array modifications. The `$unwind` aggregation stage flattens arrays into individual documents for grouping and analytics.
