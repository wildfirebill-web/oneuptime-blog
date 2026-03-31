# How to Use $merge to Upsert Aggregation Results in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Upsert

Description: Learn how MongoDB's $merge stage writes aggregation results into a collection with flexible merge strategies, supporting upserts, updates, and incremental materialized views.

---

## What Is the $merge Stage?

The `$merge` stage writes the documents produced by an aggregation pipeline into a specified collection. Unlike `$out` which replaces the entire collection, `$merge` provides fine-grained control with configurable strategies for what to do when a document already exists (matched) and when it doesn't (unmatched). It is the key tool for maintaining materialized views and incremental aggregations.

## Basic Syntax

```javascript
db.collection.aggregate([
  { ... pipeline stages ... },
  {
    $merge: {
      into: "<targetCollection>" | { db: "<db>", coll: "<collection>" },
      on: "<fieldOrFields>",
      whenMatched: "replace" | "keepExisting" | "merge" | "fail" | [<pipeline>],
      whenNotMatched: "insert" | "discard" | "fail"
    }
  }
])
```

## Default Behavior

The simplest form uses the `_id` as the merge key.

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$amount" }
    }
  },
  { $merge: { into: "customerSummary" } }
])
```

By default: replaces matched documents, inserts unmatched ones.

## Incremental Materialized View Pattern

Append or update running totals without reprocessing the entire dataset.

```javascript
db.orders.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date("2024-12-01") }
    }
  },
  {
    $group: {
      _id: "$customerId",
      monthlyTotal: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  {
    $merge: {
      into: "customerMonthly",
      on: "_id",
      whenMatched: [
        {
          $set: {
            monthlyTotal: { $add: ["$monthlyTotal", "$$new.monthlyTotal"] },
            orderCount: { $add: ["$orderCount", "$$new.orderCount"] }
          }
        }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

`$$new` refers to the incoming document from the pipeline. This is how you accumulate totals incrementally.

## Keeping Existing Documents Unchanged

```javascript
db.products.aggregate([
  { ... },
  {
    $merge: {
      into: "productSnapshot",
      whenMatched: "keepExisting",
      whenNotMatched: "insert"
    }
  }
])
// Inserts new products but doesn't overwrite existing snapshot data
```

## Replacing Matched Documents

```javascript
db.sales.aggregate([
  { ... },
  {
    $merge: {
      into: "salesReport",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## Failing on Conflicts

```javascript
db.events.aggregate([
  { ... },
  {
    $merge: {
      into: "eventArchive",
      whenMatched: "fail",
      whenNotMatched: "insert"
    }
  }
])
// Errors if any document already exists - useful for ensuring no duplicates
```

## Writing to Another Database

```javascript
db.orders.aggregate([
  { ... },
  {
    $merge: {
      into: { db: "analytics", coll: "orderSummary" },
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## $merge vs $out

| Feature | $merge | $out |
|---------|--------|------|
| Replaces entire collection | No | Yes |
| Supports upserts | Yes | No |
| Custom merge logic | Yes (pipeline) | No |
| Writes to another db | Yes | Yes |

Use `$merge` for incremental updates. Use `$out` for full replacement.

## Summary

The `$merge` stage is the most powerful way to persist aggregation results in MongoDB. Its configurable `whenMatched` and `whenNotMatched` behaviors support full replacement, incremental accumulation, conditional updates, and conflict detection. It is the foundation of materialized view patterns in MongoDB.
