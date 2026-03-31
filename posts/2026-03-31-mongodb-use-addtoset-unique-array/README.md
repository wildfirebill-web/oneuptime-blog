# How to Use $addToSet to Add Unique Elements to an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Operator, Update Operator, Document

Description: Learn how MongoDB's $addToSet operator appends a value to an array only if it is not already present, preventing duplicates in a single atomic operation.

---

## What Is the $addToSet Operator?

The `$addToSet` operator adds a value to an array field only if the value is not already present in that array. Unlike `$push`, which always appends regardless of duplicates, `$addToSet` enforces uniqueness at the database level without requiring a separate check. If the field does not exist, `$addToSet` creates it with the value as the sole element.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  { $addToSet: { <arrayField>: <value> } }
)
```

## Example: Adding a Unique Tag

```javascript
// Document: { _id: "article-1", tags: ["mongodb", "nosql"] }

db.articles.updateOne(
  { _id: "article-1" },
  { $addToSet: { tags: "database" } }
)
// tags becomes: ["mongodb", "nosql", "database"]

// Try to add "mongodb" again - no change
db.articles.updateOne(
  { _id: "article-1" },
  { $addToSet: { tags: "mongodb" } }
)
// tags remains: ["mongodb", "nosql", "database"]
```

## Example: Managing User Roles

```javascript
db.users.updateOne(
  { _id: "user-5" },
  { $addToSet: { roles: "editor" } }
)
```

This ensures the `editor` role is only added once even if the update is called multiple times, making it idempotent.

## Adding Multiple Unique Values with $each

Combine `$addToSet` with `$each` to add several values at once, skipping any that already exist.

```javascript
db.articles.updateOne(
  { _id: "article-1" },
  {
    $addToSet: {
      tags: { $each: ["mongodb", "aggregation", "tutorial"] }
    }
  }
)
// Only "aggregation" and "tutorial" are added; "mongodb" is already present
```

## Object Element Uniqueness

For objects, MongoDB checks for exact equality on the entire document, not just one field.

```javascript
// These are treated as different elements
{ $addToSet: { items: { id: 1, name: "apple" } } }
{ $addToSet: { items: { id: 1, name: "orange" } } }
```

Both would be added because the full object differs.

## Creating the Array Field If It Does Not Exist

```javascript
// Document has no "permissions" field
db.roles.updateOne(
  { _id: "admin" },
  { $addToSet: { permissions: "read" } }
)
// permissions is created: ["read"]
```

## Bulk Unique Appends

```javascript
db.subscriptions.updateMany(
  { plan: "free" },
  { $addToSet: { features: "basic-analytics" } }
)
```

## $addToSet vs $push

| Feature | $addToSet | $push |
|---------|-----------|-------|
| Prevents duplicates | Yes | No |
| Supports $each | Yes | Yes |
| Supports $sort / $slice | No | Yes |
| Supports $position | No | Yes |

## Summary

The `$addToSet` operator is the go-to choice when array elements must be unique. It is idempotent, atomic, and eliminates the need for application-level duplicate checks. For scenarios where you need both uniqueness and sorting or capping, consider post-processing with an aggregation or use a combination of `$pull` and `$push`.
