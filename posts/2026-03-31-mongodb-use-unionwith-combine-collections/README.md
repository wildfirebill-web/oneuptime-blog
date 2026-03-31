# How to Use $unionWith to Combine Collections in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Collection

Description: Learn how MongoDB's $unionWith stage combines documents from multiple collections into a single pipeline stream, similar to SQL's UNION ALL, enabling cross-collection aggregation.

---

## What Is the $unionWith Stage?

The `$unionWith` stage appends documents from a second collection (optionally processed through its own sub-pipeline) to the current pipeline's document stream. It is the MongoDB equivalent of SQL's `UNION ALL` - it combines all documents from both sources without deduplication. This is useful for querying across multiple collections that share a schema.

## Basic Syntax

```javascript
db.collection.aggregate([
  { ... pipeline stages ... },
  {
    $unionWith: {
      coll: "<secondCollection>",
      pipeline: [<optional sub-pipeline stages>]
    }
  }
])

// Shorthand (no sub-pipeline)
db.collection.aggregate([
  { $unionWith: "<secondCollection>" }
])
```

## Example: Combining Active and Archived Orders

```javascript
db.activeOrders.aggregate([
  { $match: { status: "completed" } },
  {
    $unionWith: {
      coll: "archivedOrders",
      pipeline: [
        { $match: { status: "completed" } }
      ]
    }
  },
  {
    $group: {
      _id: "$customerId",
      totalRevenue: { $sum: "$amount" }
    }
  }
])
// Aggregates completed orders from both active and archived collections
```

## Merging Logs from Multiple Services

```javascript
db.webLogs.aggregate([
  { $match: { level: "error" } },
  {
    $unionWith: {
      coll: "apiLogs",
      pipeline: [{ $match: { level: "error" } }]
    }
  },
  {
    $unionWith: {
      coll: "workerLogs",
      pipeline: [{ $match: { level: "error" } }]
    }
  },
  { $sort: { timestamp: -1 } },
  { $limit: 100 }
])
// Combined error logs from three services, most recent first
```

## Adding a Source Tag Before Combining

Use `$addFields` in the sub-pipeline to tag documents with their source collection.

```javascript
db.salesNorthAmerica.aggregate([
  { $addFields: { region: "NA" } },
  {
    $unionWith: {
      coll: "salesEurope",
      pipeline: [{ $addFields: { region: "EU" } }]
    }
  },
  {
    $unionWith: {
      coll: "salesAsia",
      pipeline: [{ $addFields: { region: "APAC" } }]
    }
  },
  {
    $group: {
      _id: "$region",
      totalRevenue: { $sum: "$amount" }
    }
  }
])
```

## Cross-Collection Full-Text Union

```javascript
db.articles.aggregate([
  { $match: { $text: { $search: "kubernetes" } } },
  {
    $unionWith: {
      coll: "tutorials",
      pipeline: [{ $match: { $text: { $search: "kubernetes" } } }]
    }
  },
  { $sort: { score: { $meta: "textScore" } } },
  { $limit: 20 }
])
```

## Simulating UNION (Deduplication)

`$unionWith` is a UNION ALL - it does not remove duplicates. To deduplicate, add a `$group` stage after.

```javascript
db.setA.aggregate([
  { $unionWith: "setB" },
  {
    $group: {
      _id: "$_id",
      doc: { $first: "$$ROOT" }
    }
  },
  { $replaceRoot: { newRoot: "$doc" } }
])
```

## Combining Collections Across Databases

```javascript
db.localUsers.aggregate([
  {
    $unionWith: {
      coll: "enterpriseUsers",
      pipeline: [
        { $match: { active: true } }
      ]
    }
  }
])
// Both collections must be in the same database
```

## $unionWith Limitations

- Both collections must be in the same database
- The sub-pipeline within `$unionWith` cannot use `$out`, `$merge`, or `$changeStream`
- Documents are combined without deduplication (UNION ALL semantics)

## Summary

The `$unionWith` stage enables cross-collection aggregation by merging document streams from multiple collections. It supports sub-pipelines for per-collection filtering and transformation before the merge. Use it to consolidate partitioned data, combine sharded collections by time period, or build unified views across service-specific collections.
