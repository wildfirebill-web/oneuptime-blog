# How to Use $match in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Performance

Description: Learn how to use the $match stage in MongoDB aggregation pipelines to filter documents early, use indexes, and improve pipeline performance.

---

## What Is the $match Stage?

The `$match` stage filters documents in an aggregation pipeline, passing only the documents that satisfy the specified condition to the next stage. Its syntax is identical to the query filter used in `find()`. Placing `$match` as early as possible in the pipeline is one of the most important performance optimizations you can make.

## Basic Syntax

```javascript
db.collection.aggregate([
  { $match: { <field>: <condition> } }
])
```

## Example: Filtering by a Single Field

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
```

Only completed orders flow into the `$group` stage.

## Example: Multiple Conditions

```javascript
db.products.aggregate([
  {
    $match: {
      category: "electronics",
      price: { $lt: 500 },
      inStock: true
    }
  },
  { $sort: { price: 1 } }
])
```

## Using Comparison Operators

```javascript
db.transactions.aggregate([
  {
    $match: {
      amount: { $gte: 1000 },
      createdAt: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2025-01-01")
      }
    }
  }
])
```

## Using $or and $and

```javascript
db.users.aggregate([
  {
    $match: {
      $or: [
        { plan: "pro" },
        { plan: "enterprise" }
      ]
    }
  }
])
```

## Matching on Array Fields

```javascript
db.articles.aggregate([
  { $match: { tags: { $in: ["mongodb", "database"] } } }
])
```

## Matching After a $group Stage

You can place `$match` after other stages to filter aggregated results.

```javascript
db.orders.aggregate([
  { $group: { _id: "$customerId", orderCount: { $sum: 1 } } },
  { $match: { orderCount: { $gte: 5 } } }
])
```

This finds customers with 5 or more orders.

## Performance: Why $match Should Come First

When `$match` is the first pipeline stage and its conditions reference an indexed field, MongoDB can use the index to avoid a full collection scan.

```javascript
// Efficient: $match uses index on createdAt
db.events.aggregate([
  { $match: { createdAt: { $gte: startDate } } },
  { $group: { _id: "$type", count: { $sum: 1 } } }
])
```

If `$match` comes after `$unwind` or `$group`, no index is available for the filter.

## $match with Text Search

```javascript
db.articles.aggregate([
  { $match: { $text: { $search: "mongodb aggregation" } } },
  { $project: { title: 1, score: { $meta: "textScore" } } },
  { $sort: { score: { $meta: "textScore" } } }
])
```

## Comparing $match in Aggregation vs find()

Both `$match` and `find()` accept the same query syntax. Use `$match` when you need further pipeline stages; use `find()` for simple queries without transformation.

## Summary

The `$match` stage is the gateway filter in MongoDB aggregation pipelines. Place it first to leverage indexes and minimize the data flowing through subsequent stages. It accepts the full MongoDB query language including comparison operators, logical operators, and text search, making it as flexible as a standard `find()` query.
