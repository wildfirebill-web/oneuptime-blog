# How to Use $unset to Remove Fields in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $unset, Pipeline Stage, Document Transformation, NoSQL

Description: Learn how to use the MongoDB $unset aggregation stage to cleanly remove fields from documents flowing through a pipeline without affecting other fields.

---

## What Is the $unset Aggregation Stage?

The `$unset` stage in aggregation pipelines removes one or more fields from documents. It is the aggregation counterpart to the `$unset` update operator, but used within `aggregate()` pipelines to strip fields from documents as they flow through stages.

```javascript
// Remove a single field
{ $unset: "fieldName" }

// Remove multiple fields
{ $unset: ["field1", "field2", "field3"] }
```

## Comparison with $project

`$unset` is a convenient shorthand for exclusion-only `$project`. These are equivalent:

```javascript
// Using $project to exclude
{ $project: { password: 0, internalId: 0 } }

// Using $unset - cleaner for exclusion-only operations
{ $unset: ["password", "internalId"] }
```

Unlike `$project`, `$unset` cannot be mixed with field inclusion - it only removes fields.

## Basic Example

Remove sensitive fields before returning user documents:

```javascript
db.users.aggregate([
  { $match: { active: true } },
  { $unset: ["password", "salt", "resetToken", "internalFlags"] }
])
```

All other fields are preserved automatically.

## Removing Nested Fields

Use dot notation to remove nested fields:

```javascript
db.orders.aggregate([
  {
    $unset: [
      "customer.ssn",
      "payment.cardNumber",
      "payment.cvv"
    ]
  }
])
```

This removes `ssn` from the nested `customer` object and `cardNumber`, `cvv` from `payment`, while keeping all other nested fields intact.

## Chaining with $addFields

A common pattern is to add computed fields and remove the source fields in the same pipeline:

```javascript
db.users.aggregate([
  {
    $addFields: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      initials: {
        $concat: [
          { $substrCP: ["$firstName", 0, 1] },
          { $substrCP: ["$lastName", 0, 1] }
        ]
      }
    }
  },
  {
    $unset: ["firstName", "lastName"]
  }
])
```

## Practical Use Case - API Response Sanitization

Strip internal fields before sending data to an external consumer:

```javascript
db.products.aggregate([
  { $match: { published: true } },
  {
    $addFields: {
      id: { $toString: "$_id" }
    }
  },
  {
    $unset: ["_id", "__v", "internalSku", "costPrice", "supplierNotes"]
  }
])
```

## Removing Array Element Fields

`$unset` works on fields within array elements when combined with `$map`:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      items: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            $unsetField: {
              field: "internalCost",
              input: "$$item"
            }
          }
        }
      }
    }
  }
])
```

For simple top-level array field removal, use `$unset` directly.

## Using $unset in Pipeline-Style Updates

`$unset` can also appear in updateMany pipeline syntax:

```javascript
db.users.updateMany(
  {},
  [
    { $unset: ["deprecatedField", "oldMetadata"] }
  ]
)
```

## Order Matters in Pipelines

Place `$unset` after stages that depend on the fields being removed:

```javascript
db.orders.aggregate([
  // Group uses costPrice - must come before $unset
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: "$price" },
      totalCost: { $sum: "$costPrice" }
    }
  },
  // Now it's safe to unset - but $group already changed the shape
  // In this case $unset would target the new grouped fields
  { $unset: ["totalCost"] }
])
```

## Summary

The `$unset` aggregation stage provides a clean, readable way to remove fields from documents in a pipeline. It is ideal for stripping sensitive fields before API responses, cleaning up intermediate fields after computations, and simplifying documents before writing them to output collections. Its simple string or array syntax makes it more readable than equivalent `$project` exclusion patterns.
