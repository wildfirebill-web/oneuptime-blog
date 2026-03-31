# How to Use $in to Match Any Value in a List in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $in, Filter, List Match

Description: Learn how to use MongoDB's $in operator to match documents where a field equals any value from a specified list, replacing multiple $or conditions.

---

## What Is $in

The `$in` operator matches documents where the field value equals any value in a specified array. It is the MongoDB equivalent of SQL's `IN (val1, val2, val3)`.

Syntax:

```javascript
{ field: { $in: [value1, value2, value3] } }
```

## Basic Example

Find users with specific roles:

```javascript
db.users.find({ role: { $in: ["admin", "editor", "moderator"] } })
```

This returns every user whose `role` field is `"admin"`, `"editor"`, or `"moderator"`.

## $in vs Multiple $or Conditions

The following two queries are logically equivalent, but `$in` is more concise and typically more efficient:

```javascript
// Using $or - verbose
db.products.find({
  $or: [
    { status: "active" },
    { status: "featured" },
    { status: "sale" }
  ]
})

// Using $in - clean and equivalent
db.products.find({ status: { $in: ["active", "featured", "sale"] } })
```

## Matching Against Array Fields

When the field contains an array, `$in` matches if any element in the field array is in the specified list:

```javascript
// Matches documents where tags contains "mongodb" or "database"
db.posts.find({ tags: { $in: ["mongodb", "database"] } })
```

## Using $in with ObjectIds

A very common use case is fetching multiple documents by their IDs:

```javascript
const { ObjectId } = require("mongodb");

const ids = [
  new ObjectId("64a1b2c3d4e5f6789abc0001"),
  new ObjectId("64a1b2c3d4e5f6789abc0002"),
  new ObjectId("64a1b2c3d4e5f6789abc0003")
];

const docs = await db.collection("products").find({
  _id: { $in: ids }
}).toArray();
```

## Using $in with Regex Patterns

`$in` can also contain regular expressions:

```javascript
db.users.find({
  email: { $in: [/@gmail\.com$/, /@yahoo\.com$/] }
})
```

## Index Support

`$in` queries are index-friendly. MongoDB performs an index lookup for each value in the array. Keep the list reasonably sized (under a few hundred values) for best performance.

```javascript
await db.collection("orders").createIndex({ status: 1 });

// Efficient - uses index for each status value
db.orders.find({ status: { $in: ["pending", "processing"] } })
```

## Combining $in with Other Conditions

```javascript
db.orders.find({
  status: { $in: ["pending", "processing"] },
  createdAt: { $gte: new Date("2025-01-01") },
  total: { $gt: 100 }
})
```

## Common Mistakes

- Passing a very large array (thousands of values) to `$in` - this is slow and may exceed document size limits. Consider a different query strategy.
- Confusing `$in` (field equals any listed value) with `$all` (array field contains all listed values).
- Expecting `$in: []` (empty array) to match anything - it matches nothing.

## Summary

The `$in` operator efficiently matches documents where a field equals any value from a provided list. It is cleaner than multiple `$or` conditions and is well-supported by indexes. Use it for status filters, ID lookups, and tag matching, but avoid passing very large lists as this degrades performance.
