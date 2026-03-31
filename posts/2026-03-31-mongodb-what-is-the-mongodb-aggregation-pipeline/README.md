# What Is the MongoDB Aggregation Pipeline

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation Pipeline, Analytics, Query, Data Processing

Description: Learn what the MongoDB aggregation pipeline is, how it processes documents through sequential stages, and how to use common stages for analytics and data transformation.

---

## What Is the Aggregation Pipeline

The MongoDB aggregation pipeline processes documents through a sequence of stages. Each stage transforms the documents passing through it. The output of one stage becomes the input of the next, similar to Unix pipes.

## Basic Pipeline Structure

```javascript
db.collection.aggregate([
  { $stage1: { ... } },
  { $stage2: { ... } },
  { $stage3: { ... } }
])
```

## Core Aggregation Stages

### $match - Filter Documents

Similar to `find()` query conditions. Put this first to reduce data early:

```javascript
{ $match: { status: "completed", total: { $gte: 100 } } }
```

### $group - Aggregate Data

Group documents by a key and apply accumulators:

```javascript
{
  $group: {
    _id: "$category",
    totalSales: { $sum: "$amount" },
    avgSale: { $avg: "$amount" },
    count: { $sum: 1 },
    maxSale: { $max: "$amount" }
  }
}
```

### $project - Reshape Documents

Include, exclude, rename, or compute fields:

```javascript
{
  $project: {
    _id: 0,
    name: 1,
    fullName: { $concat: ["$firstName", " ", "$lastName"] },
    yearJoined: { $year: "$createdAt" }
  }
}
```

### $sort - Sort Documents

```javascript
{ $sort: { totalSales: -1, category: 1 } }
```

### $limit and $skip

```javascript
{ $limit: 20 }
{ $skip: 40 }
```

### $lookup - Join Collections

```javascript
{
  $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }
}
```

### $unwind - Flatten Arrays

```javascript
// Unwind turns one document with an array into multiple documents
{ $unwind: "$tags" }
```

### $addFields - Add Computed Fields

```javascript
{
  $addFields: {
    totalWithTax: { $multiply: ["$total", 1.1] },
    month: { $month: "$createdAt" }
  }
}
```

## Complete Example: Monthly Sales Report

```javascript
db.orders.aggregate([
  // Stage 1: Filter completed orders from last year
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: new Date("2025-01-01"),
        $lt: new Date("2026-01-01")
      }
    }
  },
  // Stage 2: Group by month
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      revenue: { $sum: "$total" },
      orders: { $sum: 1 },
      avgOrderValue: { $avg: "$total" }
    }
  },
  // Stage 3: Sort chronologically
  { $sort: { "_id.year": 1, "_id.month": 1 } },
  // Stage 4: Format output
  {
    $project: {
      _id: 0,
      month: {
        $dateToString: {
          format: "%Y-%m",
          date: {
            $dateFromParts: {
              year: "$_id.year",
              month: "$_id.month"
            }
          }
        }
      },
      revenue: { $round: ["$revenue", 2] },
      orders: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  }
])
```

## Performance Best Practices

```javascript
// Always put $match before $group to reduce data early
// Use indexes - $match and $sort use indexes when placed at the beginning
// Use $project to reduce document size before expensive stages
// Allow disk use for large aggregations:

db.orders.aggregate(pipeline, { allowDiskUse: true })
```

## Summary

The MongoDB aggregation pipeline processes documents through sequential stages where each stage transforms the data. Core stages include `$match` for filtering, `$group` for aggregation, `$project` for reshaping, `$lookup` for joins, and `$sort` for ordering. Place `$match` early in the pipeline and leverage indexes to minimize the data processed in each subsequent stage.
