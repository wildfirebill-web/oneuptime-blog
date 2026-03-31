# How to Use $eq for Exact Match Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $eq, Exact Match, Filter

Description: Learn how to use MongoDB's $eq operator for exact match queries, when it is implicit, and how it differs from direct equality syntax.

---

## What Is $eq

The `$eq` operator matches documents where a field's value is exactly equal to the specified value. It is the explicit form of the equality condition.

Syntax:

```javascript
{ field: { $eq: value } }
```

## Implicit vs Explicit $eq

In most cases, MongoDB's shorthand equality syntax `{ field: value }` is equivalent to `{ field: { $eq: value } }`. Both of the following queries return the same results:

```javascript
// Implicit equality (shorthand)
db.users.find({ status: "active" })

// Explicit $eq
db.users.find({ status: { $eq: "active" } })
```

The explicit form is useful in programmatic query construction where you're building filter objects dynamically.

## Matching Different Data Types

`$eq` is type-sensitive. The compared value must match both value and BSON type.

```javascript
// Matches documents where age is the number 30
db.users.find({ age: { $eq: 30 } })

// Does NOT match documents where age is the string "30"
db.users.find({ age: { $eq: "30" } })
```

## Matching Nested Fields

Use dot notation to match fields inside embedded documents:

```javascript
db.orders.find({ "address.country": { $eq: "US" } })
```

## Matching Array Elements

When applied to an array field, `$eq` matches any document where the array contains the specified value as an element:

```javascript
// Matches docs where tags array contains "mongodb"
db.posts.find({ tags: { $eq: "mongodb" } })

// Equivalent shorthand
db.posts.find({ tags: "mongodb" })
```

## Matching an Entire Array

To match a document where an array field is exactly equal to a specific array (same elements, same order):

```javascript
db.items.find({ colors: { $eq: ["red", "blue"] } })
```

This only matches if `colors` is exactly `["red", "blue"]` - same elements and same order.

## Using $eq in Aggregation Pipelines

`$eq` in aggregation is a comparison expression that returns a boolean:

```javascript
db.orders.aggregate([
  {
    $project: {
      isHighValue: { $eq: ["$status", "completed"] },
      orderId: 1,
      total: 1
    }
  }
])
```

## $eq with Indexes

`$eq` queries are index-friendly. If the queried field has an index, MongoDB uses it for an efficient index scan:

```javascript
await db.collection("users").createIndex({ email: 1 });

// Uses index efficiently
db.users.find({ email: { $eq: "alice@example.com" } })
```

## Common Mistakes

- Comparing a number to a string version of the number - types must match.
- Expecting array equality to be order-independent (use `$all` for that).
- Using `$eq` inside aggregation `$match` - the shorthand `{ field: value }` is simpler.

## Summary

The `$eq` operator performs an exact equality match in MongoDB queries. In most query contexts, the shorthand `{ field: value }` is equivalent and more concise. Use the explicit `$eq` form when building dynamic query objects programmatically. Remember that comparisons are type-sensitive, and for array fields `$eq` checks whether the array contains the value as an element.
