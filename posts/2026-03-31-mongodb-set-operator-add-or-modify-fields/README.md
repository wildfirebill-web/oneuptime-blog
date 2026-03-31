# How to Use $set to Add or Modify Fields in MongoDB Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $set, Update Operator, Document, Write Operation

Description: Learn how to use MongoDB's $set update operator to add new fields or modify existing fields without replacing the entire document.

---

## What Is $set

The `$set` operator sets the value of a field in a document. If the field exists, it updates it. If the field does not exist, `$set` creates it. This is the most commonly used update operator because it performs partial updates without affecting other fields.

Syntax:

```javascript
{ $set: { field1: value1, field2: value2, ... } }
```

## Basic Examples

```javascript
// Update status and set a timestamp
db.orders.updateOne(
  { _id: orderId },
  { $set: { status: "shipped", shippedAt: new Date() } }
)
```

If `shippedAt` did not exist before, it is created. If `status` already existed, it is overwritten.

## Adding New Fields

`$set` is the standard way to add a field to documents:

```javascript
// Add a new "tier" field to all users
await db.collection("users").updateMany(
  {},
  { $set: { tier: "free" } }
)
```

## Updating Nested Fields

Use dot notation to target fields inside embedded documents:

```javascript
db.users.updateOne(
  { _id: userId },
  { $set: { "address.city": "San Francisco", "address.zip": "94102" } }
)
```

This updates only the `city` and `zip` subfields of `address`, leaving the rest of `address` unchanged.

## Setting Fields in Array Elements

Use the positional `$` operator to update a specific matched array element:

```javascript
// Update the status of the first item that matches the filter
db.orders.updateOne(
  { _id: orderId, "items.sku": "SKU001" },
  { $set: { "items.$.status": "packed" } }
)
```

Use `$[]` to update all elements in an array:

```javascript
// Mark all items in the order as reviewed
db.orders.updateOne(
  { _id: orderId },
  { $set: { "items.$[].reviewed": true } }
)
```

## Setting Multiple Fields at Once

```javascript
await db.collection("profiles").updateOne(
  { userId: "u42" },
  {
    $set: {
      "personal.firstName": "Alice",
      "personal.lastName": "Smith",
      "preferences.theme": "dark",
      updatedAt: new Date()
    }
  }
)
```

## $set vs $setOnInsert

`$setOnInsert` is like `$set` but only applies when an upsert creates a new document:

```javascript
await db.collection("users").updateOne(
  { email: "alice@example.com" },
  {
    $set: { lastLogin: new Date() },
    $setOnInsert: { createdAt: new Date(), role: "user" }
  },
  { upsert: true }
)
```

`createdAt` and `role` are only set when the document is inserted, not when it is updated.

## $set in Aggregation Update Pipelines

In MongoDB 4.2+, you can use an aggregation pipeline in update operations:

```javascript
await db.collection("users").updateMany(
  {},
  [
    {
      $set: {
        fullName: { $concat: ["$firstName", " ", "$lastName"] },
        updatedAt: "$$NOW"
      }
    }
  ]
)
```

This allows computed field values using aggregation expressions.

## Common Mistakes

- Writing `{ field: value }` directly as the update document instead of `{ $set: { field: value } }` - the bare document replaces the entire document.
- Using `$set` on an array field with a scalar value - it replaces the entire array.
- Forgetting dot notation for nested fields - `{ $set: { address: { city: "NYC" } } }` replaces the entire `address` subdocument.

## Summary

`$set` is the go-to update operator for adding or modifying fields in MongoDB documents without affecting other fields. Use dot notation for nested fields, `$` or `$[]` for array elements, and aggregation pipelines for computed values. Always use `$set` (rather than a bare replacement document) when you want partial updates.
