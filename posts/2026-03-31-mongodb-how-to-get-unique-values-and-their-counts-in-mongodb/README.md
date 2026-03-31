# How to Get Unique Values and Their Counts in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $group, Unique Values, Distinct

Description: Learn how to get unique field values and their occurrence counts in MongoDB using $group, distinct(), and aggregation pipelines with practical examples.

---

## Introduction

Getting unique values and their counts is a common analytics task in MongoDB. You can use the `distinct()` method for simple unique value lists, or the aggregation pipeline with `$group` and `$sum` for full value-count breakdowns. Aggregations are more powerful and support sorting, filtering, and field combinations.

## Simple Distinct Values (No Counts)

The `distinct()` method returns a list of unique values for a field:

```javascript
// Get all distinct values of the status field
db.orders.distinct("status");
// Returns: ["pending", "processing", "completed", "cancelled"]

// Distinct with a query filter
db.orders.distinct("status", { customerId: ObjectId("...") });
```

## Unique Values with Counts Using $group

Use `$group` with `$sum: 1` to count occurrences of each unique value:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
]);
```

Output:

```javascript
{ _id: "completed", count: 4521 }
{ _id: "pending", count: 1203 }
{ _id: "processing", count: 892 }
{ _id: "cancelled", count: 445 }
```

## Rename the _id Field in Output

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      status: "$_id",
      count: 1
    }
  },
  { $sort: { count: -1 } }
]);
```

## Multiple Field Combinations

Get unique combinations of two fields and their counts:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { region: "$region", status: "$status" },
      count: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      region: "$_id.region",
      status: "$_id.status",
      count: 1
    }
  },
  { $sort: { region: 1, count: -1 } }
]);
```

## Top N Values by Count

Get the top 5 most common product categories:

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 5 }
]);
```

## Unique Values from an Array Field

For fields that are arrays, use `$unwind` before grouping:

```javascript
// Get unique tags and how many posts use each tag
db.blogPosts.aggregate([
  { $unwind: "$tags" },
  {
    $group: {
      _id: "$tags",
      postCount: { $sum: 1 }
    }
  },
  { $sort: { postCount: -1 } }
]);
```

## Percentage Distribution

Calculate the percentage share of each value:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  },
  {
    $group: {
      _id: null,
      total: { $sum: "$count" },
      statuses: { $push: { status: "$_id", count: "$count" } }
    }
  },
  { $unwind: "$statuses" },
  {
    $project: {
      _id: 0,
      status: "$statuses.status",
      count: "$statuses.count",
      percentage: {
        $round: [
          { $multiply: [{ $divide: ["$statuses.count", "$total"] }, 100] },
          1
        ]
      }
    }
  },
  { $sort: { count: -1 } }
]);
```

## Filtered Unique Value Counts

Get unique values for a filtered subset of documents:

```javascript
db.orders.aggregate([
  { $match: { createdAt: { $gte: new Date("2026-01-01") } } },
  {
    $group: {
      _id: "$paymentMethod",
      count: { $sum: 1 },
      totalRevenue: { $sum: "$amount" }
    }
  },
  { $sort: { count: -1 } }
]);
```

## Unique Values with Additional Aggregations

Combine counts with other metrics per unique value:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      orderCount: { $sum: 1 },
      totalSpent: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" },
      lastOrderDate: { $max: "$createdAt" }
    }
  },
  { $sort: { totalSpent: -1 } },
  { $limit: 100 }
]);
```

## Summary

Use `db.collection.distinct()` for a simple list of unique values, and `$group` with `$sum: 1` in aggregation pipelines for full value-count breakdowns. Aggregation pipelines are far more flexible - they support filtering with `$match`, multi-field grouping, array unwinding with `$unwind`, percentage calculations, and combining counts with other aggregated metrics like sums and averages in a single pipeline.
