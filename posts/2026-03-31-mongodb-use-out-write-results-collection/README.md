# How to Use $out to Write Aggregation Results to a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Materialized View

Description: Learn how to use MongoDB's $out stage to write aggregation pipeline results into a new or existing collection, replacing its contents completely with the output.

---

## What Is the $out Stage?

The `$out` stage writes the documents produced by an aggregation pipeline into a specified collection. It must be the last stage in the pipeline. If the target collection already exists, `$out` replaces its entire contents with the new results. If the collection does not exist, it creates it. This makes `$out` ideal for full-refresh materialized views and pre-computed aggregation tables.

## Basic Syntax

```javascript
db.collection.aggregate([
  { ... pipeline stages ... },
  { $out: "<targetCollectionName>" }
])

// Writing to another database
db.collection.aggregate([
  { ... pipeline stages ... },
  { $out: { db: "<databaseName>", coll: "<collectionName>" } }
])
```

## Example: Creating a Daily Sales Summary

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        date: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
        region: "$region"
      },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.date": -1 } },
  { $out: "dailySalesSummary" }
])
// The dailySalesSummary collection is replaced with the latest results
```

## Example: Pre-computing Top Products

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalSold: { $sum: "$items.quantity" },
      revenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
    }
  },
  { $sort: { revenue: -1 } },
  { $limit: 100 },
  { $out: "topProducts" }
])
```

## Scheduling Regular Refreshes

For reporting tables, run `$out` on a schedule.

```javascript
// Run this as a cron job or scheduled task
async function refreshMonthlySummary() {
  await db.orders.aggregate([
    { $match: { status: "completed" } },
    {
      $group: {
        _id: { $month: "$createdAt" },
        total: { $sum: "$amount" }
      }
    },
    { $out: "monthlySummary" }
  ]).toArray()
}
```

## Atomicity Behavior

`$out` is atomic at the collection level. The old collection is not replaced until all pipeline results are written. If the pipeline fails, the original collection is unchanged.

```javascript
// Safe: the original collection survives pipeline errors
db.bigTable.aggregate([
  { $group: { _id: "$key", count: { $sum: 1 } } },
  { $out: "bigTableSummary" }
])
// If any stage errors, bigTableSummary retains its old content
```

## Writing to Another Database

```javascript
db.rawEvents.aggregate([
  { $match: { processed: false } },
  { $group: { _id: "$type", count: { $sum: 1 } } },
  { $out: { db: "analytics", coll: "eventSummary" } }
])
```

## Limitations

- `$out` replaces the entire target collection - it does not support incremental updates
- Cannot write to a collection that is the source of the aggregation
- If the target collection has existing indexes, they are preserved after the replacement
- Cannot be used inside a `$facet` sub-pipeline

## $out vs $merge

| Feature | $out | $merge |
|---------|------|--------|
| Full collection replacement | Yes | No |
| Incremental upserts | No | Yes |
| Custom conflict handling | No | Yes |
| Use case | Full refresh | Incremental update |

## Creating a Permanent View vs $out

For live data, use `db.createView()`. For pre-computed snapshots that don't need to be live, use `$out`.

```javascript
// $out creates a real collection with a snapshot in time
db.orders.aggregate([...pipeline..., { $out: "orderSnapshot" }])

// createView creates a virtual collection re-evaluated on each read
db.createView("orderView", "orders", [...pipeline...])
```

## Summary

The `$out` stage persists aggregation results into a MongoDB collection, fully replacing it on each run. It is the right tool for refreshing pre-computed reporting tables and materialized views on a schedule. For incremental updates that merge new results with existing data, use `$merge` instead.
