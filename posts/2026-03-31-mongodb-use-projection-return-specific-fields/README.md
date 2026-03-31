# How to Use Projection to Return Only Specific Fields in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Projection, Query, Performance, Field Selection

Description: Learn how to use MongoDB projection to return only the fields you need, reducing network overhead and improving query performance.

---

## What Is Projection in MongoDB

When you run a `find()` query in MongoDB, by default the server returns every field in every matching document. For documents with many fields, this wastes bandwidth and memory. Projection lets you tell MongoDB exactly which fields to include or exclude in the result set.

Projection is the second argument to `find()`, and it is a document where each key is a field name and each value is `1` (include) or `0` (exclude).

## Including Specific Fields

To return only certain fields, set their values to `1` in the projection document. MongoDB always returns `_id` unless you explicitly exclude it.

```javascript
db.users.find(
  { status: "active" },
  { name: 1, email: 1 }
)
```

This returns only `_id`, `name`, and `email` for each matching user.

To also suppress `_id`:

```javascript
db.users.find(
  { status: "active" },
  { name: 1, email: 1, _id: 0 }
)
```

## Excluding Specific Fields

To return everything except certain fields, set their values to `0`. You cannot mix inclusion and exclusion in the same projection (except for `_id`).

```javascript
db.users.find(
  { status: "active" },
  { password: 0, internalNotes: 0 }
)
```

This returns all fields except `password` and `internalNotes`.

## Projecting Nested Fields

Use dot notation to project fields inside embedded documents.

```javascript
db.orders.find(
  { status: "shipped" },
  { "address.city": 1, "address.zip": 1, total: 1 }
)
```

Only the `city` and `zip` subfields of `address` are returned, along with `total`.

## Projection with the Node.js Driver

In the Node.js MongoDB driver, pass the projection as part of the options object:

```javascript
const results = await db.collection("users").find(
  { status: "active" },
  { projection: { name: 1, email: 1, _id: 0 } }
).toArray();
```

## Projection and Index Coverage

When the fields you project are all part of an index, MongoDB can satisfy the query entirely from the index without reading the actual documents. This is called a covered query and is significantly faster.

```javascript
// Assume index: { status: 1, name: 1, email: 1 }
db.users.find(
  { status: "active" },
  { name: 1, email: 1, _id: 0 }
)
```

Use `explain("executionStats")` to confirm `IXSCAN` with no `FETCH` stage.

## Common Mistakes

- Mixing inclusion and exclusion fields (other than `_id`) causes an error.
- Forgetting `_id: 0` when you do not want the `_id` field in results.
- Using projection on array fields without understanding `$slice` or `$elemMatch` for finer control.

## Summary

MongoDB projection is specified as the second argument to `find()` and lets you include or exclude specific fields from query results. Use inclusion projections (`field: 1`) to return only the fields you need, and exclusion projections (`field: 0`) to strip sensitive or bulky fields. When projected fields match an index, MongoDB can serve a covered query for maximum performance.
