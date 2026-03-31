# How to Use $merge to Update a Collection from Aggregation Results in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $merge, Pipeline, Collection

Description: Use the MongoDB $merge aggregation stage to write aggregation results directly into a target collection, enabling upserts, merges, and incremental updates.

---

## What is $merge

The `$merge` stage is the final stage in an aggregation pipeline that writes output documents to a specified collection. Unlike `$out` which replaces the entire target collection, `$merge` can insert, update, merge, or keep existing documents based on a matching key.

## Basic Syntax

```javascript
db.sourceCollection.aggregate([
  { $match: { status: "completed" } },
  { $group: {
    _id: "$region",
    totalRevenue: { $sum: "$amount" },
    orderCount: { $count: {} }
  }},
  { $merge: {
    into: "regional_summary",
    on: "_id",
    whenMatched: "merge",
    whenNotMatched: "insert"
  }}
])
```

## Use Case 1: Building a Summary Collection

Aggregate raw orders into a daily summary:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        date: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
        productId: "$productId"
      },
      dailyRevenue: { $sum: "$amount" },
      unitsSold: { $sum: "$quantity" }
    }
  },
  {
    $project: {
      date: "$_id.date",
      productId: "$_id.productId",
      dailyRevenue: 1,
      unitsSold: 1,
      updatedAt: "$$NOW"
    }
  },
  {
    $merge: {
      into: "daily_product_summary",
      on: ["date", "productId"],
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## Use Case 2: Incrementally Updating Totals

Use a pipeline in `whenMatched` to increment existing values rather than replacing them:

```javascript
db.events.aggregate([
  {
    $match: {
      timestamp: {
        $gte: ISODate("2026-03-31T00:00:00Z"),
        $lt: ISODate("2026-04-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: "$userId",
      newEvents: { $count: {} },
      newPoints: { $sum: "$points" }
    }
  },
  {
    $merge: {
      into: "user_totals",
      on: "_id",
      whenMatched: [
        {
          $set: {
            totalEvents: { $add: ["$totalEvents", "$$new.newEvents"] },
            totalPoints: { $add: ["$totalPoints", "$$new.newPoints"] },
            lastUpdated: "$$NOW"
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

The `$$new` variable refers to the incoming document from the pipeline.

## Use Case 3: Syncing a Denormalized Cache Collection

Refresh a materialized view nightly:

```javascript
db.products.aggregate([
  {
    $lookup: {
      from: "inventory",
      localField: "_id",
      foreignField: "productId",
      as: "stock"
    }
  },
  {
    $set: {
      availableStock: { $sum: "$stock.quantity" },
      lastSynced: "$$NOW"
    }
  },
  {
    $unset: "stock"
  },
  {
    $merge: {
      into: "product_cache",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## $merge vs $out Comparison

```text
Feature             | $merge           | $out
--------------------|------------------|------------------
Preserves existing  | Yes              | No (replaces all)
Partial updates     | Yes              | No
Pipeline whenMatched| Yes              | No
Target must exist   | No (auto-creates)| No (auto-creates)
Atomic              | Per document     | Entire collection
```

## Summary

The `$merge` aggregation stage writes pipeline results to a target collection with flexible conflict resolution: `whenMatched` controls what happens when a document with the matching key already exists (replace, merge, keepExisting, or a custom pipeline), while `whenNotMatched` controls inserts. This makes `$merge` ideal for building and maintaining summary tables, materialized views, and cached aggregation results without full collection replacement.
