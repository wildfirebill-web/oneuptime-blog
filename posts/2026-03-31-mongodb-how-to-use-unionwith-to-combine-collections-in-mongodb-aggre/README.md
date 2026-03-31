# How to Use $unionWith to Combine Collections in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $unionWith, Collection Merging, Pipeline Stages, NoSQL

Description: Learn how to use MongoDB's $unionWith stage to combine documents from multiple collections into a single pipeline result, similar to SQL UNION ALL.

---

## What Is the $unionWith Stage?

The `$unionWith` stage (introduced in MongoDB 4.4) performs a union of the current pipeline's documents with documents from another collection. It is the MongoDB equivalent of SQL's `UNION ALL` - it combines all documents from both sources without deduplication.

```javascript
{
  $unionWith: {
    coll: "otherCollection",
    pipeline: [/* optional pipeline to apply to the other collection */]
  }
}
```

Or simply:

```javascript
{ $unionWith: "otherCollection" }
```

## Basic Example - Combining Two Collections

Combine `activeOrders` and `archivedOrders` into a single result:

```javascript
db.activeOrders.aggregate([
  {
    $unionWith: "archivedOrders"
  }
])
```

Returns all documents from both collections.

## Combining with Matching Fields

For a clean union, ensure both collections have compatible fields:

```javascript
db.salesQ1.aggregate([
  { $project: { amount: 1, date: 1, region: 1, _id: 0 } },
  {
    $unionWith: {
      coll: "salesQ2",
      pipeline: [
        { $project: { amount: 1, date: 1, region: 1, _id: 0 } }
      ]
    }
  }
])
```

## Aggregating Across Multiple Collections

Combine collections and then aggregate the merged result:

```javascript
db.ordersJanuary.aggregate([
  {
    $unionWith: {
      coll: "ordersFebruary",
      pipeline: []
    }
  },
  {
    $unionWith: {
      coll: "ordersMarch",
      pipeline: []
    }
  },
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalSpent: -1 } }
])
```

## Filtering the Unioned Collection

Apply a pipeline to the secondary collection before the union:

```javascript
db.products.aggregate([
  { $match: { category: "Electronics" } },
  {
    $unionWith: {
      coll: "newArrivals",
      pipeline: [
        { $match: { category: "Electronics" } },
        { $addFields: { isNew: true } }
      ]
    }
  },
  { $sort: { name: 1 } }
])
```

## Cross-Database Union

Union collections from different databases:

```javascript
db.getSiblingDB("db1").customers.aggregate([
  {
    $unionWith: {
      coll: "customers",
      db: "db2"
    }
  }
])
```

## Practical Use Case - Multi-Tenant Data Aggregation

Each tenant has their own collection; aggregate across all:

```javascript
db.tenant_acme_events.aggregate([
  { $addFields: { tenant: "acme" } },
  {
    $unionWith: {
      coll: "tenant_globex_events",
      pipeline: [{ $addFields: { tenant: "globex" } }]
    }
  },
  {
    $unionWith: {
      coll: "tenant_initech_events",
      pipeline: [{ $addFields: { tenant: "initech" } }]
    }
  },
  {
    $group: {
      _id: { tenant: "$tenant", eventType: "$type" },
      count: { $sum: 1 }
    }
  }
])
```

## Practical Use Case - Time-Partitioned Collections

Many applications partition data by time period. `$unionWith` enables querying across partitions:

```javascript
db.logs_2024_01.aggregate([
  {
    $unionWith: {
      coll: "logs_2024_02",
      pipeline: []
    }
  },
  {
    $unionWith: {
      coll: "logs_2024_03",
      pipeline: []
    }
  },
  { $match: { level: "ERROR", service: "api-gateway" } },
  { $sortByCount: "$errorCode" }
])
```

## Deduplication After Union

Since `$unionWith` is UNION ALL (not UNION DISTINCT), use `$group` to deduplicate:

```javascript
db.collectionA.aggregate([
  { $unionWith: "collectionB" },
  {
    $group: {
      _id: "$uniqueKey",
      // Keep fields from one or both documents
      doc: { $first: "$$ROOT" }
    }
  },
  { $replaceRoot: { newRoot: "$doc" } }
])
```

## Restrictions

- `$unionWith` cannot reference a sharded collection as the secondary collection (the primary can be sharded).
- The secondary collection's pipeline has the same stage restrictions as any aggregation pipeline.
- `$out` and `$merge` cannot be used inside the `$unionWith` pipeline.

## Summary

The `$unionWith` stage enables SQL-UNION-style combining of multiple collections in a single aggregation pipeline. It is invaluable for time-partitioned data, multi-tenant aggregations, and scenarios where related data is spread across collections. Apply pipelines to each collection within `$unionWith` to normalize and filter data before the merge, then aggregate the combined result downstream.
