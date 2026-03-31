# How to Create a Wildcard Index for Dynamic Schemas in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Wildcard Index, Dynamic Schema, Database

Description: Learn how to create wildcard indexes in MongoDB to index all fields or a subset of fields in collections with dynamic or unpredictable schema structures.

---

## Overview

Wildcard indexes (introduced in MongoDB 4.2) allow you to index all fields in a document, or all fields within a sub-document, without knowing the field names in advance. They are designed for collections with highly dynamic or variable schemas where the set of queryable fields changes across documents.

## Creating a Wildcard Index on All Fields

Use `$**` to create an index covering every field in every document:

```javascript
db.userAttributes.createIndex({ "$**": 1 })
```

This creates index entries for every field path in every document, enabling queries on any field.

## Querying with a Wildcard Index

After creating the wildcard index, any equality or range query on any field can potentially use it:

```javascript
// These can all use the wildcard index
db.userAttributes.find({ "preferences.theme": "dark" })
db.userAttributes.find({ "customField1": "value" })
db.userAttributes.find({ "nested.deeply.field": { $gt: 10 } })
```

## Creating a Wildcard Index on a Specific Sub-Document

Instead of indexing all fields, index only fields within a specific embedded document:

```javascript
db.products.createIndex({ "attributes.$**": 1 })
```

This indexes all fields within the `attributes` sub-document:

```javascript
// These queries can use the index
db.products.find({ "attributes.color": "red" })
db.products.find({ "attributes.size": "XL" })
db.products.find({ "attributes.material": "cotton" })
```

## Including and Excluding Fields in Wildcard Indexes

Use `wildcardProjection` to include or exclude specific fields:

```javascript
// Include only specific fields
db.products.createIndex(
  { "$**": 1 },
  {
    wildcardProjection: {
      "attributes": 1,
      "metadata": 1
    }
  }
)

// Exclude specific fields from the wildcard index
db.products.createIndex(
  { "$**": 1 },
  {
    wildcardProjection: {
      "internalData": 0,
      "largeTextField": 0
    }
  }
)
```

Note: You can only include OR exclude fields in a single `wildcardProjection` (not both, except `_id`).

## Checking Wildcard Index Usage

Verify which fields are covered by the wildcard index:

```javascript
db.userAttributes.find({ "preferences.language": "en" }).explain("executionStats")
```

Look for `IXSCAN` in the winning plan and verify the index name includes `$**`.

## Wildcard Index Limitations

Important constraints to understand:

```text
- Wildcard indexes do NOT support:
  - Compound indexes (only single-field wildcard)
  - $text, $near, $geoWithin queries
  - Covered queries (cannot use wildcard index for covered projections)
  - Unique, TTL, sparse, or hashed options

- Performance considerations:
  - Each field path creates a separate index entry
  - Large documents with many fields increase index size significantly
  - Write overhead is proportional to the number of indexed fields per document
```

## When to Use Wildcard Indexes

```text
Good use cases:
- Collections with user-defined or schema-flexible attributes
- Product catalogs with category-specific attributes
- Multi-tenant applications where tenants define custom fields
- Development and prototyping when schema is not yet fixed

Poor use cases:
- Collections with stable, known schemas (use targeted indexes)
- Fields requiring unique constraints
- High-write-throughput collections with many fields per document
```

## Practical Example: Product Attribute Search

```javascript
// Product documents have varying attributes per category
// Electronics: { "attributes.wattage": 65, "attributes.voltage": 110 }
// Clothing: { "attributes.size": "M", "attributes.color": "blue" }
// Books: { "attributes.isbn": "978-...", "attributes.genre": "fiction" }

db.products.createIndex({ "attributes.$**": 1 })

// All of these can now use the index
db.products.find({ "attributes.size": "M" })
db.products.find({ "attributes.wattage": { $lte: 100 } })
db.products.find({ "attributes.genre": "fiction" })
```

## Summary

Wildcard indexes in MongoDB solve the challenge of querying dynamically structured documents without requiring a separate index for every possible field. By indexing all fields (or all fields within a sub-document path), they enable flexible ad-hoc querying on unknown field names. They are ideal for product attribute tables, user-defined metadata, and multi-tenant attribute stores, but carry a write overhead proportional to document field count. For collections with known, stable schemas, targeted indexes remain more efficient.
