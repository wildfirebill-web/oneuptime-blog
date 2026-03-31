# How to Build a Sales Funnel Analysis in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Funnel, Pipeline

Description: Learn how to build a sales funnel analysis in MongoDB using the aggregation pipeline to track user progression through conversion stages.

---

## Introduction

A sales funnel analysis measures how many users progress through each stage of a defined conversion flow - for example, from viewing a product to adding it to cart, checking out, and completing a purchase. MongoDB's aggregation pipeline can compute stage-level counts and conversion rates in a single query.

## Sample Data Structure

Assume a `funnel_events` collection tracking user actions:

```json
{
  "_id": ObjectId("..."),
  "userId": "user_101",
  "stage": "view_product",
  "sessionId": "sess_abc",
  "timestamp": ISODate("2025-06-10T09:15:00Z")
}
```

Possible stage values: `view_product`, `add_to_cart`, `checkout`, `purchase`.

## Counting Users at Each Stage

Group by stage and count distinct users who reached it:

```javascript
db.funnel_events.aggregate([
  {
    $match: {
      timestamp: {
        $gte: ISODate("2025-06-01T00:00:00Z"),
        $lt: ISODate("2025-07-01T00:00:00Z")
      },
      stage: { $in: ["view_product", "add_to_cart", "checkout", "purchase"] }
    }
  },
  {
    $group: {
      _id: "$stage",
      uniqueUsers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      stage: "$_id",
      count: { $size: "$uniqueUsers" }
    }
  }
])
```

This gives you the raw user counts per stage, but stages arrive in arbitrary order. Sort them in application code or use a `$sortArray` expression in MongoDB 5.2+.

## Enforcing Stage Order with $facet

To get all stages in a single structured document with conversion rates, use `$facet` to compute each stage independently:

```javascript
db.funnel_events.aggregate([
  {
    $match: {
      timestamp: { $gte: ISODate("2025-06-01T00:00:00Z"), $lt: ISODate("2025-07-01T00:00:00Z") }
    }
  },
  {
    $facet: {
      view_product: [
        { $match: { stage: "view_product" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ],
      add_to_cart: [
        { $match: { stage: "add_to_cart" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ],
      checkout: [
        { $match: { stage: "checkout" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ],
      purchase: [
        { $match: { stage: "purchase" } },
        { $group: { _id: null, users: { $addToSet: "$userId" } } },
        { $project: { count: { $size: "$users" } } }
      ]
    }
  }
])
```

## Computing Drop-Off Rates in Application Code

After retrieving stage counts, compute drop-off rates in your application:

```javascript
function buildFunnel(result) {
  const stages = ["view_product", "add_to_cart", "checkout", "purchase"];
  let topCount = null;
  return stages.map(stage => {
    const count = result[stage][0]?.count ?? 0;
    if (topCount === null) topCount = count;
    return {
      stage,
      count,
      conversionFromTop: topCount > 0 ? (count / topCount * 100).toFixed(1) + "%" : "0%"
    };
  });
}
```

## Index for Funnel Queries

```javascript
db.funnel_events.createIndex({ timestamp: 1, stage: 1, userId: 1 })
```

## Summary

Building a sales funnel in MongoDB involves using `$match` to filter by time period and stage, `$group` with `$addToSet` to count unique users per stage, and optionally `$facet` to compute all stages in parallel. Conversion rates and drop-off percentages are typically calculated in application code after retrieving the per-stage counts. Index on `timestamp`, `stage`, and `userId` to keep these aggregations fast.
