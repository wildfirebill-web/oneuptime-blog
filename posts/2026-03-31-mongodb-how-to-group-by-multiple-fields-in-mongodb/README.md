# How to Group by Multiple Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $group, Multiple Fields, Aggregation, Compound Grouping

Description: Learn how to group MongoDB documents by multiple fields simultaneously using the $group aggregation stage with a compound _id object.

---

## Overview

MongoDB's `$group` stage supports grouping by multiple fields by passing a compound object as the `_id`. This creates one group for each unique combination of the specified field values.

## Basic Multi-Field Grouping

```javascript
// Group by status AND category
db.orders.aggregate([
  {
    $group: {
      _id: {
        status: "$status",
        category: "$category"
      },
      count: { $sum: 1 },
      totalAmount: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.status": 1, "_id.category": 1 } }
]);

// Result
// { "_id": { "status": "shipped", "category": "electronics" }, "count": 42, "totalAmount": 8400 }
// { "_id": { "status": "shipped", "category": "clothing" }, "count": 17, "totalAmount": 680 }
```

## Group by Region and Year

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: {
        region: "$region",
        year: { $year: "$date" }
      },
      totalRevenue: { $sum: "$revenue" },
      salesCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": -1, "_id.region": 1 } }
]);
```

## Renaming the Output Fields

Use `$project` after `$group` to flatten the `_id` object into top-level fields:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { status: "$status", region: "$region" },
      count: { $sum: 1 },
      totalAmount: { $sum: "$amount" }
    }
  },
  {
    $project: {
      _id: 0,
      status: "$_id.status",
      region: "$_id.region",
      count: 1,
      totalAmount: 1
    }
  },
  { $sort: { region: 1, status: 1 } }
]);
```

## Group by Date Parts (Year, Month, Day)

```javascript
// Order counts by year and month
db.orders.aggregate([
  {
    $group: {
      _id: {
        year:  { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      orderCount: { $sum: 1 },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } },
  {
    $project: {
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-",
          { $toString: "$_id.month" }
        ]
      },
      orderCount: 1,
      revenue: 1,
      _id: 0
    }
  }
]);
```

## Group by Three Fields

```javascript
// Group by country, state, and product category
db.sales.aggregate([
  {
    $group: {
      _id: {
        country:  "$country",
        state:    "$state",
        category: "$productCategory"
      },
      units: { $sum: "$unitsSold" },
      revenue: { $sum: "$revenue" }
    }
  },
  { $match: { revenue: { $gt: 1000 } } }
]);
```

## Multi-Field Group with Multiple Accumulators

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        customerId: "$customerId",
        status: "$status"
      },
      count: { $sum: 1 },
      totalSpent: { $sum: "$amount" },
      avgOrder: { $avg: "$amount" },
      lastOrderDate: { $max: "$createdAt" }
    }
  }
]);
```

## Creating a Pivot-Like Structure

Group by one field and collect another as an array:

```javascript
// Get all statuses for each customer as an array
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      statuses: { $addToSet: "$status" },
      orderCount: { $sum: 1 }
    }
  }
]);
```

## Node.js Example

```javascript
async function getRevenueByRegionAndCategory(db) {
  return db.collection("orders").aggregate([
    { $match: { status: "completed" } },
    {
      $group: {
        _id: { region: "$region", category: "$category" },
        revenue: { $sum: "$amount" },
        orders: { $sum: 1 }
      }
    },
    {
      $project: {
        _id: 0,
        region: "$_id.region",
        category: "$_id.category",
        revenue: { $round: ["$revenue", 2] },
        orders: 1
      }
    },
    { $sort: { region: 1, revenue: -1 } }
  ]).toArray();
}
```

## Summary

To group MongoDB documents by multiple fields, pass a compound object as the `$group` `_id`, with each key mapping to a field path using `$fieldName` syntax. This creates one group per unique combination of all specified fields. Use `$project` after `$group` to flatten the nested `_id` object into top-level fields for cleaner output. Date part extraction operators (`$year`, `$month`, etc.) allow grouping by time dimensions within the same multi-field group expression.
