# How to Use $set to Add or Modify Fields in MongoDB Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operator, $set, Document Updates, NoSQL

Description: Learn how to use MongoDB's $set update operator to add new fields or modify existing fields in documents without affecting other fields.

---

## What Is the $set Operator?

The `$set` operator is the most commonly used update operator in MongoDB. It sets the value of a field in a document. If the field does not exist, `$set` creates it. Existing fields not mentioned in `$set` remain unchanged.

```javascript
db.collection.updateOne(
  { filter },
  { $set: { field1: value1, field2: value2 } }
)
```

## Basic Usage

Given a `users` collection:

```javascript
{ _id: 1, name: "Alice", age: 28, city: "New York" }
```

Update the city and add a new field:

```javascript
db.users.updateOne(
  { _id: 1 },
  { $set: { city: "San Francisco", lastLogin: new Date() } }
)
```

Result:

```javascript
{ _id: 1, name: "Alice", age: 28, city: "San Francisco", lastLogin: ISODate("...") }
```

The `name` and `age` fields were not affected.

## Updating Nested Fields

Use dot notation to update nested fields:

```javascript
db.users.updateOne(
  { _id: 1 },
  { $set: { "address.street": "123 Main St", "address.zip": "94102" } }
)
```

If `address` doesn't exist, MongoDB creates the nested structure automatically.

## Updating Array Elements by Index

```javascript
// Update the second element (index 1) of the scores array
db.students.updateOne(
  { _id: 1 },
  { $set: { "scores.1": 95 } }
)
```

## Updating Multiple Documents

Use `updateMany` to update all matching documents:

```javascript
db.products.updateMany(
  { category: "Electronics" },
  { $set: { inStock: true, updatedAt: new Date() } }
)
```

## Setting Fields to Specific Types

```javascript
// Set a field to an array
db.users.updateOne({ _id: 1 }, { $set: { roles: ["user", "moderator"] } })

// Set a field to a nested document
db.orders.updateOne(
  { _id: 101 },
  { $set: { shippingAddress: { street: "456 Oak Ave", city: "Austin" } } }
)

// Set a field to null (field remains but is null)
db.sessions.updateOne({ _id: "sess1" }, { $set: { userId: null } })
```

## Upsert with $set

When using `upsert: true`, `$set` sets fields on both insert and update:

```javascript
db.configs.updateOne(
  { key: "theme" },
  { $set: { value: "dark", updatedAt: new Date() } },
  { upsert: true }
)
```

If no document with `key: "theme"` exists, MongoDB inserts one with both the filter fields and the `$set` fields.

## Combining $set with $setOnInsert

Use `$setOnInsert` alongside `$set` to set different fields on insert vs. update:

```javascript
db.configs.updateOne(
  { key: "theme" },
  {
    $set: { value: "dark", updatedAt: new Date() },
    $setOnInsert: { createdAt: new Date() }
  },
  { upsert: true }
)
```

## Replacing vs. $set

Avoid using `replaceOne` when you only need to update some fields. `$set` is safer because it preserves other fields:

```javascript
// DANGEROUS - replaces the entire document
db.users.replaceOne({ _id: 1 }, { name: "Alice Updated" })
// This removes age, city, and all other fields!

// SAFE - only updates the name field
db.users.updateOne({ _id: 1 }, { $set: { name: "Alice Updated" } })
```

## Summary

The `$set` operator is the primary tool for adding and modifying fields in MongoDB documents. It supports dot notation for nested updates, array index targeting, and works seamlessly with upsert operations. Always prefer `$set` over full document replacement when updating individual fields to avoid unintentional data loss.
