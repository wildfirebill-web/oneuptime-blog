# How to Use $group to Aggregate and Summarize Data in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Group By

Description: Learn how to use MongoDB's $group stage to group documents by key fields and compute aggregates like sum, average, count, min, and max across grouped documents.

---

## What Is the $group Stage?

The `$group` stage groups documents by a specified expression and applies accumulator operators to compute aggregate values across each group. It is the MongoDB equivalent of SQL's `GROUP BY` and is one of the most powerful stages in the aggregation pipeline.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $group: {
      _id: <groupKeyExpression>,
      <field>: { <accumulator>: <expression> }
    }
  }
])
```

The `_id` field is required and defines the grouping key. Set it to `null` to aggregate across all documents.

## Example: Count Documents per Category

```javascript
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 }
    }
  }
])
```

## Example: Sum Revenue per Customer

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  }
])
```

## Common Accumulator Operators

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      total: { $sum: "$revenue" },
      average: { $avg: "$revenue" },
      highest: { $max: "$revenue" },
      lowest: { $min: "$revenue" },
      count: { $sum: 1 }
    }
  }
])
```

## Grouping by Multiple Fields

Pass an object to `_id` to group by a combination of fields.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { year: { $year: "$createdAt" }, status: "$status" },
      total: { $sum: "$amount" }
    }
  }
])
```

## Counting All Documents (No Grouping)

Set `_id` to `null` to compute a single aggregate over all documents.

```javascript
db.users.aggregate([
  {
    $group: {
      _id: null,
      totalUsers: { $sum: 1 },
      avgAge: { $avg: "$age" }
    }
  }
])
```

## Collecting Array Values with $push

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      orderIds: { $push: "$_id" }
    }
  }
])
```

## Collecting Unique Values with $addToSet

```javascript
db.logs.aggregate([
  {
    $group: {
      _id: "$userId",
      uniqueActions: { $addToSet: "$action" }
    }
  }
])
```

## Grouping by Date Components

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        month: { $month: "$timestamp" },
        year: { $year: "$timestamp" }
      },
      eventCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

## Filtering Groups with $match After $group

```javascript
db.orders.aggregate([
  { $group: { _id: "$customerId", orderCount: { $sum: 1 } } },
  { $match: { orderCount: { $gte: 10 } } }
])
```

## Summary

The `$group` stage is the foundation of data aggregation in MongoDB. It groups documents by any expression, supports a rich set of accumulators for sums, averages, extremes, and array collection, and pairs naturally with `$match` before it to filter input and after it to filter aggregated results.
