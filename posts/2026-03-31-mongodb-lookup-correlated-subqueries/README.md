# How to Use Correlated Subqueries with $lookup in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Database

Description: Learn how to write correlated subqueries in MongoDB using the pipeline form of $lookup with let variables to join and filter related collections dynamically.

---

A correlated subquery is one where the inner query references values from the outer query. In MongoDB, the pipeline form of `$lookup` - introduced in MongoDB 3.6 - enables correlated subqueries by letting you pass outer document fields into the inner pipeline through `let` variables.

## Syntax

```javascript
db.collection.aggregate([
  {
    $lookup: {
      from: "otherCollection",
      let: { outerField: "$fieldName" },
      pipeline: [
        { $match: { $expr: { $eq: ["$innerField", "$$outerField"] } } },
        // additional pipeline stages
      ],
      as: "resultField"
    }
  }
])
```

`let` captures outer fields as variables. Inside the pipeline, reference them with `$$variableName`. Use `$expr` inside `$match` to compare fields across the join.

## Basic Correlated Join - Orders with Recent Items

Join each order to its line items, but only include items created in the last 30 days:

```javascript
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)

db.orders.aggregate([
  {
    $lookup: {
      from: "lineItems",
      let: { orderId: "$_id" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$orderId", "$$orderId"] },
                { $gte: ["$createdAt", thirtyDaysAgo] }
              ]
            }
          }
        }
      ],
      as: "recentItems"
    }
  }
])
```

The `$$orderId` variable threads the outer `_id` into the inner `$match`, making it a correlated lookup.

## Filtering Joined Documents - Users with Active Subscriptions

For each user, fetch only their currently active subscriptions:

```javascript
db.users.aggregate([
  {
    $lookup: {
      from: "subscriptions",
      let: { userId: "$_id", now: new Date() },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$userId", "$$userId"] },
                { $lte: ["$startDate", "$$now"] },
                { $gte: ["$endDate", "$$now"] }
              ]
            }
          }
        },
        { $project: { plan: 1, endDate: 1 } }
      ],
      as: "activeSubscriptions"
    }
  },
  { $match: { "activeSubscriptions.0": { $exists: true } } }
])
```

The final `$match` filters out users with no active subscriptions.

## Aggregating Inside the Subquery Pipeline

Compute the total spend per customer by aggregating inside the correlated pipeline:

```javascript
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      let: { customerId: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$customerId", "$$customerId"] } } },
        { $group: { _id: null, totalSpend: { $sum: "$amount" } } }
      ],
      as: "spendSummary"
    }
  },
  {
    $addFields: {
      totalSpend: { $ifNull: [{ $arrayElemAt: ["$spendSummary.totalSpend", 0] }, 0] }
    }
  }
])
```

## Correlated Subquery vs. Simple Join

| Form | Use When |
|---|---|
| `localField` / `foreignField` | Simple equality join, no filtering inside |
| `let` + `pipeline` | Need conditions, transformations, or aggregation inside the joined set |

## Index Tip

Create an index on the joined collection's filter field to avoid collection scans for each outer document:

```javascript
db.lineItems.createIndex({ orderId: 1, createdAt: -1 })
```

## Summary

The pipeline form of `$lookup` with `let` variables enables correlated subqueries in MongoDB aggregation. It lets you reference outer document values inside the joined collection's pipeline, supporting conditional joins, date-range filtering, and even aggregation within the subquery. Always index the correlated join field on the inner collection to keep performance efficient.
