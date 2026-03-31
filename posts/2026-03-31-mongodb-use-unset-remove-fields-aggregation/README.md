# How to Use $unset to Remove Fields in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Field Removal

Description: Learn how to use the $unset stage in MongoDB aggregation pipelines to remove one or more fields from documents before they reach the next stage or the output.

---

## What Is the $unset Stage?

The `$unset` stage removes one or more fields from documents in an aggregation pipeline. It is the aggregation equivalent of excluding fields in `$project`, but with a cleaner syntax when the goal is purely removal. All other fields in the document are preserved.

## Basic Syntax

```javascript
// Remove a single field
db.collection.aggregate([
  { $unset: "<fieldName>" }
])

// Remove multiple fields
db.collection.aggregate([
  { $unset: ["<field1>", "<field2>"] }
])
```

## Example: Removing a Single Field

```javascript
db.users.aggregate([
  { $unset: "password" }
])
// All user documents are returned without the password field
```

## Example: Removing Multiple Fields

```javascript
db.users.aggregate([
  { $unset: ["password", "internalNotes", "legacyId"] }
])
```

## Removing Nested Fields

Use dot notation to remove a field inside an embedded document.

```javascript
db.orders.aggregate([
  { $unset: "payment.cardNumber" }
])
```

## Combining $unset with Other Stages

```javascript
db.products.aggregate([
  { $match: { category: "electronics" } },
  {
    $addFields: {
      finalPrice: { $multiply: ["$price", 0.95] }
    }
  },
  { $unset: ["costPrice", "internalSku"] }
])
```

This pipeline filters, adds a computed field, then removes internal fields before returning results.

## $unset in Multi-Stage Pipelines for Security

Use `$unset` to strip sensitive fields from results exposed to the client.

```javascript
db.employees.aggregate([
  { $match: { department: "engineering" } },
  { $unset: ["salary", "ssn", "homeAddress"] }
])
```

## $unset vs $project for Field Removal

Both achieve field removal, but differ in verbosity:

```javascript
// Using $project - must list all fields to keep
db.users.aggregate([
  { $project: { name: 1, email: 1, _id: 1 } }
])

// Using $unset - specify only what to remove
db.users.aggregate([
  { $unset: ["password", "securityQuestion"] }
])
```

`$unset` is more maintainable when documents have many fields and you only want to remove a few.

## Removing Array-Level Fields

`$unset` can remove fields from within array elements using dot notation and combined with `$map` if needed.

```javascript
db.orders.aggregate([
  {
    $addFields: {
      items: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            $unsetField: { field: "costPrice", input: "$$item" }
          }
        }
      }
    }
  }
])
```

For removing fields inside arrays, `$unsetField` expression is more appropriate than the `$unset` stage.

## $unset as an Update Stage

In update pipelines, `$unset` can also remove fields from persisted documents.

```javascript
db.users.updateMany(
  {},
  [{ $unset: "legacyToken" }]
)
```

This uses the aggregation pipeline form of `updateMany` to run `$unset` as an update.

## Summary

The `$unset` stage provides a concise and readable way to remove fields from documents in MongoDB aggregation pipelines. It is ideal for stripping sensitive or internal fields before returning results, and is more maintainable than listing all retained fields in `$project` when only a few fields need to be removed.
