# How to Use the aggregate Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline, Analytics

Description: Learn how to use MongoDB's aggregate command to build multi-stage data transformation pipelines with $match, $group, $lookup, $project, and $sort stages.

---

## The Aggregation Pipeline

The `aggregate()` command processes documents through a sequence of stages. Each stage transforms the data and passes it to the next. This approach is more powerful and performant than JavaScript-based `mapReduce`.

```javascript
db.collection.aggregate([
  { stage1 },
  { stage2 },
  ...
]);
```

## $match - Filtering Documents

`$match` should be the first stage whenever possible to reduce documents early and leverage indexes:

```javascript
db.orders.aggregate([
  { $match: { status: "completed", total: { $gt: 100 } } }
]);
```

## $group - Aggregating Data

`$group` groups documents by a key and computes accumulated values:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: "$total" },
      orderCount:   { $sum: 1 },
      avgOrder:     { $avg: "$total" },
      maxOrder:     { $max: "$total" }
    }
  },
  { $sort: { totalRevenue: -1 } }
]);
```

## $project - Reshaping Documents

`$project` includes, excludes, and computes new fields:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      discountedPrice: { $multiply: ["$price", 0.9] },
      _id: 0
    }
  }
]);
```

## $lookup - Joining Collections

`$lookup` performs a left outer join with another collection:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  {
    $project: {
      orderId: 1,
      total: 1,
      "customer.name": 1,
      "customer.email": 1
    }
  }
]);
```

## $unwind - Flattening Arrays

`$unwind` deconstructs an array field, creating one document per array element:

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalQtySold: { $sum: "$items.qty" }
    }
  }
]);
```

## $addFields - Adding Computed Fields

```javascript
db.orders.aggregate([
  {
    $addFields: {
      tax:   { $multiply: ["$subtotal", 0.08] },
      total: { $add: ["$subtotal", { $multiply: ["$subtotal", 0.08] }] }
    }
  }
]);
```

## $limit and $skip for Pagination

```javascript
db.products.aggregate([
  { $match: { inStock: true } },
  { $sort: { price: 1 } },
  { $skip: 20 },
  { $limit: 10 }
]);
```

## allowDiskUse for Large Datasets

Aggregations are limited to 100 MB of memory by default. For larger datasets:

```javascript
db.events.aggregate(
  [
    { $group: { _id: "$userId", events: { $push: "$$ROOT" } } }
  ],
  { allowDiskUse: true }
);
```

## Summary

MongoDB's `aggregate()` command builds data transformation pipelines with stages like `$match` (filter), `$group` (aggregate), `$project` (reshape), `$lookup` (join), and `$unwind` (flatten arrays). Always place `$match` first to use indexes, use `$addFields` for computed values, and set `allowDiskUse: true` for pipelines that process large datasets beyond the 100 MB memory limit.
