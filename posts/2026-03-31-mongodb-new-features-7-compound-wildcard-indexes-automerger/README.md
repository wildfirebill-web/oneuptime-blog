# How to Use New Features in MongoDB 7.0 (Compound Wildcard Indexes, AutoMerger)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Wildcard Index, AutoMerger, Feature, Version

Description: Explore MongoDB 7.0's key new features including compound wildcard indexes for dynamic schemas and the AutoMerger for sharded clusters.

---

## Introduction

MongoDB 7.0 introduced compound wildcard indexes, the AutoMerger for automated chunk consolidation in sharded clusters, improved `$lookup` performance, and enhanced time series collection capabilities. This post covers the two most impactful features for schema flexibility and sharding operations.

## Compound Wildcard Indexes

MongoDB 6.0 introduced wildcard indexes, but they could not be combined with other fields in a single index. MongoDB 7.0 lifts this restriction, allowing you to create compound indexes that include both specific field paths and a wildcard component.

### Creating a Compound Wildcard Index

```javascript
db.products.createIndex({
  category: 1,
  "attributes.$**": 1
})
```

This index covers queries that filter on `category` plus any field within the `attributes` subdocument.

### Use Case: Dynamic Attribute Schemas

Compound wildcard indexes are particularly useful when documents have a fixed set of common fields plus a variable set of attributes:

```json
{
  "_id": ObjectId("..."),
  "category": "Electronics",
  "name": "Bluetooth Speaker",
  "attributes": {
    "color": "black",
    "batteryLife": 20,
    "bluetooth": "5.0"
  }
}
```

A query like `{ category: "Electronics", "attributes.color": "black" }` can now use the compound wildcard index without requiring separate indexes for each attribute.

### Checking Index Usage

```javascript
db.products.find({
  category: "Electronics",
  "attributes.batteryLife": { $gte: 10 }
}).explain("executionStats")
```

Look for `IXSCAN` on the compound wildcard index in the `winningPlan`.

### Limitations

- The wildcard component must be a suffix in the compound index (not the leading field).
- Compound wildcard indexes cannot include array fields alongside the wildcard.
- Covered queries are not supported for the wildcard portion.

## AutoMerger

In sharded clusters, chunk splitting over time creates many small chunks, increasing routing overhead and memory usage in `mongos`. MongoDB 7.0 introduces the AutoMerger background process that automatically merges adjacent chunks on the same shard when they can be safely combined.

### Checking AutoMerger Status

```javascript
db.adminCommand({ autoMergerStatus: 1 })
```

### Enabling and Disabling AutoMerger

AutoMerger is enabled by default in MongoDB 7.0. To disable it globally:

```javascript
db.adminCommand({ configureAutoMerger: 1, enable: false })
```

To disable it for a specific collection:

```javascript
sh.disableAutoMerger("mydb.orders")
```

To re-enable:

```javascript
sh.enableAutoMerger("mydb.orders")
```

### Manual Merge Trigger

If you want to trigger a merge immediately instead of waiting for the AutoMerger schedule:

```javascript
db.adminCommand({
  mergeAllChunksOnShard: "mydb.orders",
  shard: "shard01"
})
```

## Additional MongoDB 7.0 Highlights

- `$lookup` now pushes `$match` stages inside the lookup pipeline, improving join performance significantly.
- Time series collections gain secondary index support on non-time and non-metadata fields.
- `mongodump` and `mongorestore` include the `--numInsertionWorkersPerCollection` flag for faster parallel restores.

## Summary

MongoDB 7.0's compound wildcard indexes solve the dynamic-schema indexing problem by allowing a single index to cover queries across variable subdocument fields alongside fixed fields. The AutoMerger eliminates manual chunk consolidation tasks for sharded clusters by automatically merging adjacent chunks. Together these features reduce operational overhead and improve query performance for both flexible-schema and high-scale deployments.
