# How to Create a Wildcard Index for Dynamic Schemas in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Wildcard Index, Dynamic Schema, Flexible Data

Description: Learn how to create wildcard indexes in MongoDB to support queries on collections with dynamic or unpredictable field names and nested structures.

---

## Overview

Wildcard indexes in MongoDB (available since 4.2) index all fields or a subset of fields in documents. They are designed for collections with dynamic or varied schemas where you cannot predict which fields will be queried. Instead of creating individual indexes on every possible field, a single wildcard index covers them all.

## Creating a Wildcard Index on All Fields

```javascript
// Index all fields at all depths
db.products.createIndex({ "$**": 1 })
```

This creates one index that supports queries on any field in any document.

## Creating a Wildcard Index on a Specific Path

```javascript
// Index all fields within the "metadata" subdocument
db.products.createIndex({ "metadata.$**": 1 })

// Index all fields within "attributes"
db.inventory.createIndex({ "attributes.$**": 1 })
```

## Practical Example - Product Catalog with Dynamic Attributes

```javascript
db.products.insertMany([
  {
    name: "Laptop",
    category: "electronics",
    attributes: { cpu: "Intel i7", ram: "16GB", storage: "512GB SSD" }
  },
  {
    name: "T-Shirt",
    category: "clothing",
    attributes: { size: "M", color: "blue", material: "cotton" }
  },
  {
    name: "Coffee Mug",
    category: "kitchen",
    attributes: { capacity: "350ml", material: "ceramic", dishwasherSafe: true }
  }
])

// Single wildcard index covers all attribute fields
db.products.createIndex({ "attributes.$**": 1 })

// All these queries use the wildcard index
db.products.find({ "attributes.cpu": "Intel i7" })
db.products.find({ "attributes.color": "blue" })
db.products.find({ "attributes.capacity": "350ml" })
db.products.find({ "attributes.dishwasherSafe": true })
```

## Wildcard Index with wildcardProjection

Use `wildcardProjection` to include or exclude specific fields from the wildcard index:

```javascript
// Include only specific fields
db.events.createIndex(
  { "$**": 1 },
  {
    wildcardProjection: {
      "metadata.source": 1,
      "metadata.version": 1,
      "payload": 1
    }
  }
)

// Exclude specific fields (inverse)
db.logs.createIndex(
  { "$**": 1 },
  {
    wildcardProjection: {
      "_id": 0,
      "timestamp": 0,
      "rawData": 0
    }
  }
)
```

## Verifying Wildcard Index Usage

```javascript
// Check that queries use the wildcard index
db.products.find({ "attributes.color": "red" }).explain("executionStats")
// winningPlan should show IXSCAN, not COLLSCAN
// index: "$**_1" or "attributes.$**_1"
```

## When to Use Wildcard Indexes

```text
Good use cases:
- Collections storing user-defined attributes or metadata
- IoT sensor data with varying field names per device type
- CMS content with custom fields per content type
- Multi-tenant apps where tenants define their own schema
- Prototyping when schema is not finalized

Not ideal for:
- Collections with a known, stable schema (use targeted indexes)
- High-write workloads (wildcard indexes are larger and slower to update)
- Fields requiring compound indexes or range + sort optimization
```

## Wildcard Index Limitations

```text
- Cannot create a unique wildcard index
- Cannot use wildcard indexes as shard keys
- Wildcard indexes do not support $text queries
- Cannot create compound wildcard indexes (with multiple fields) before MongoDB 7.0
- Wildcard indexes cannot cover queries (they are not covering indexes)
- $or queries may not use a wildcard index efficiently
```

## Compound Wildcard Indexes (MongoDB 7.0+)

```javascript
// MongoDB 7.0+: compound wildcard index
db.data.createIndex({ tenantId: 1, "metadata.$**": 1 })

// Supports queries like:
db.data.find({ tenantId: "tenant1", "metadata.status": "active" })
```

## Checking Index Size

```javascript
db.products.stats().indexSizes
// Wildcard indexes can be large on collections with many fields
// Monitor size and compare against targeted index alternatives
```

## Summary

Wildcard indexes solve the problem of indexing dynamic or unpredictable field names by creating a single index that covers all fields (or a specified subtree) in a collection. Use `{ "$**": 1 }` for all fields or `{ "path.$**": 1 }` for a specific subdocument. Use `wildcardProjection` to narrow the scope. They are most valuable for collections with user-defined attributes or highly variable schemas, but they come with trade-offs in index size, write performance, and query optimization compared to targeted indexes.
