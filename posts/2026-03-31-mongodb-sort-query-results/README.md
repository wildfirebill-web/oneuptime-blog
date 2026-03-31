# How to Sort Query Results in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sort, Query, Index, Aggregation

Description: Learn how to sort MongoDB query results using sort(), including ascending and descending order, multi-field sorting, and index optimization.

---

## Sorting with sort()

The `sort()` method specifies the order in which MongoDB returns documents. Pass a document where each key is a field name and the value is `1` for ascending order or `-1` for descending.

```javascript
// Ascending by name
db.users.find().sort({ name: 1 })

// Descending by createdAt (newest first)
db.posts.find({ status: "published" }).sort({ createdAt: -1 })
```

## Multi-Field Sorting

You can sort by multiple fields. MongoDB applies the sort criteria left to right.

```javascript
db.products.find().sort({ category: 1, price: -1 })
```

This sorts first by `category` alphabetically, then within each category by `price` from highest to lowest.

## Sorting with limit() and skip()

Always specify `sort()` before `limit()` and `skip()` for consistent pagination:

```javascript
const recentPosts = await db.collection("posts")
  .find({ status: "published" })
  .sort({ publishedAt: -1 })
  .skip(20)
  .limit(10)
  .toArray();
```

## Index-Backed Sorting

MongoDB can sort efficiently when the sort fields are indexed. Without an index, it performs an in-memory sort that has a 100 MB memory limit by default.

```javascript
// Create index to support sort
await db.collection("orders").createIndex({ createdAt: -1 });

// This query uses the index and avoids in-memory sort
db.orders.find({ status: "shipped" }).sort({ createdAt: -1 })
```

Use `explain()` to verify the sort uses an index (`IXSCAN` without `SORT` stage):

```javascript
db.orders.find({ status: "shipped" }).sort({ createdAt: -1 }).explain("executionStats")
```

## Sorting in the Aggregation Pipeline

Use `$sort` in an aggregation pipeline:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $sort: { total: -1, createdAt: -1 } },
  { $limit: 50 }
])
```

Placing `$sort` after `$match` and before `$limit` lets MongoDB optimize by sorting only the matched set.

## Case-Insensitive Sorting with Collation

By default, MongoDB sorts strings by binary value. For case-insensitive or locale-aware sorting, use collation:

```javascript
db.users.find().sort({ name: 1 }).collation({ locale: "en", strength: 2 })
```

`strength: 2` makes the sort case-insensitive.

## Sorting by Text Score

When using full-text search, sort by relevance score:

```javascript
db.articles.find(
  { $text: { $search: "mongodb performance" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

## Common Mistakes

- Sorting on unindexed fields with large result sets, hitting the 100 MB memory limit.
- Forgetting `sort()` before `skip()` in paginated queries, leading to inconsistent pages.
- Mixing ascending and descending in multi-field sorts without a compound index covering the same order.

## Summary

Use `sort()` with `1` (ascending) or `-1` (descending) to order MongoDB query results. For large result sets, add an index matching your sort fields to avoid slow in-memory sorts. Always specify `sort()` when using `limit()` or `skip()` to ensure consistent, repeatable pagination.
