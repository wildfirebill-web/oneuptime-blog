# How to Index for Lookup (Join) Operations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Lookup, Join, Index, Aggregation

Description: Optimize MongoDB $lookup join performance by indexing foreign collection fields, using pipeline $lookup for complex conditions, and measuring lookup cost with explain.

---

## How $lookup Works

`$lookup` performs a left outer join between a local collection and a foreign collection. For each document in the local collection, MongoDB queries the foreign collection to find matching documents. Without an index on the foreign field, every `$lookup` causes a collection scan of the foreign collection.

## Basic $lookup

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "users",           // Foreign collection
      localField: "userId",   // Field in orders
      foreignField: "_id",    // Field in users to match
      as: "userInfo",
    },
  },
]);
```

## Required Index on the Foreign Collection

The index must be on the `foreignField` in the `from` collection:

```javascript
// If foreignField is _id, the default _id index already covers it
// For any other foreign field, create an index explicitly

db.products.createIndex({ sku: 1 }); // Index for joining on sku

db.orderItems.aggregate([
  {
    $lookup: {
      from: "products",
      localField: "productSku",
      foreignField: "sku",
      as: "product",
    },
  },
]);
```

## Verify Lookup Uses the Index

```javascript
db.orderItems.explain("executionStats").aggregate([
  {
    $lookup: {
      from: "products",
      localField: "productSku",
      foreignField: "sku",
      as: "product",
    },
  },
]);
```

In the explain output, check `totalDocsExamined` for the `$lookup` stage. If it equals the full `products` collection size, the index is not being used.

## Filter Before $lookup to Reduce Work

Apply `$match` before `$lookup` to minimize the number of documents being joined:

```javascript
// GOOD - only join orders from the last 7 days
db.orders.aggregate([
  { $match: { createdAt: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } } },
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user",
    },
  },
]);
```

## Pipeline $lookup for Complex Join Conditions

Use the pipeline form of `$lookup` for joins with multiple conditions or filters on the foreign collection:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "inventory",
      let: { productId: "$productId", warehouseId: "$warehouseId" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$productId", "$$productId"] },
                { $eq: ["$warehouseId", "$$warehouseId"] },
                { $gt: ["$qty", 0] },
              ],
            },
          },
        },
        { $project: { qty: 1, location: 1 } },
      ],
      as: "stock",
    },
  },
]);

// Create a compound index to support the pipeline $match
db.inventory.createIndex({ productId: 1, warehouseId: 1 });
```

## Unwind After $lookup

`$lookup` always returns an array in the `as` field. Use `$unwind` to flatten it for further processing:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user",
    },
  },
  { $unwind: "$user" }, // Flatten array to single document
  {
    $project: {
      orderId: 1,
      amount: 1,
      "user.name": 1,
      "user.email": 1,
    },
  },
]);
```

## When to Embed Instead of Lookup

`$lookup` adds latency and memory usage. Consider embedding if:
- The joined data is always accessed together with the parent
- The joined data does not change frequently
- The embedded data is bounded in size

Use `$lookup` when:
- The foreign data is shared across many documents
- The foreign data changes independently
- You need to query the foreign collection independently

## Summary

`$lookup` performance depends on an index on the `foreignField` in the joined collection - without one, every join triggers a full collection scan. Apply `$match` before `$lookup` to reduce the working set. Use the pipeline form of `$lookup` with `$expr` for multi-field join conditions and create matching compound indexes on the foreign collection. Consider embedding as an alternative when join data is always co-accessed and bounded in size.
