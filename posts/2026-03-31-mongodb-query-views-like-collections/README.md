# How to Query Views Like Regular Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, View, Query, Aggregation, Database Design

Description: Learn how to query MongoDB views using find, countDocuments, and aggregate just like regular collections, and understand the performance considerations involved.

---

MongoDB views behave like read-only collections from a query perspective. Any `find` or `aggregate` command that works on a collection also works on a view, with one key difference: the view's pipeline runs first, then your query applies on top.

## Basic find Queries

```javascript
// Assume "active_users" is a view over the "users" collection
db.active_users.find()
db.active_users.find({ plan: "pro" })
db.active_users.find({ plan: "pro" }, { name: 1, email: 1 })
```

These work exactly as they would on a regular collection.

## Sorting and Pagination

```javascript
db.active_users.find()
  .sort({ createdAt: -1 })
  .skip(20)
  .limit(10)
```

## countDocuments and estimatedDocumentCount

```javascript
db.active_users.countDocuments({ plan: "enterprise" })
```

Note: `estimatedDocumentCount()` is not available on views because views have no stored document count metadata.

## Running aggregate on a View

You can pipeline additional stages on top of a view:

```javascript
db.active_users.aggregate([
  { $match: { plan: "pro" } },
  {
    $group: {
      _id: "$country",
      userCount: { $sum: 1 }
    }
  },
  { $sort: { userCount: -1 } }
])
```

MongoDB internally composes your query/pipeline with the view's own pipeline, executing them together against the source collection.

## Using findOne

```javascript
db.active_users.findOne({ email: "alice@example.com" })
```

## Differences from Regular Collections

| Operation            | Regular Collection | View          |
|----------------------|-------------------|---------------|
| `find`               | Supported         | Supported     |
| `aggregate`          | Supported         | Supported     |
| `insert/update/delete` | Supported       | Not allowed   |
| `createIndex`        | Supported         | Not allowed   |
| `estimatedDocumentCount` | Supported    | Not available |
| `distinct`           | Supported         | Supported     |

## distinct on a View

```javascript
db.active_users.distinct("country")
```

## Performance Considerations

Views do not store data and have no indexes of their own. Query predicates added on top of a view are evaluated after the view's pipeline, which may result in collection scans unless the source collection has appropriate indexes.

Use `explain` to inspect the execution plan:

```javascript
db.active_users.find({ plan: "pro" }).explain("executionStats")
```

If the plan shows a COLLSCAN on the source collection, add an index on the source collection (not the view) to speed it up:

```javascript
// Add index on the underlying source collection
db.users.createIndex({ plan: 1, status: 1 })
```

## Summary

MongoDB views support `find`, `aggregate`, `countDocuments`, and `distinct` just like regular collections. Write operations and direct index creation are not permitted. Query performance depends on indexes on the underlying source collection - use `explain` to verify that the right indexes are being used.
