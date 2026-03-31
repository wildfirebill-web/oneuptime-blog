# How to Use $mergeObjects to Combine Objects in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Object Operators, $mergeObjects, Database

Description: Learn how to use $mergeObjects in MongoDB aggregation to combine multiple documents or sub-documents into a single merged object with field override semantics.

---

## Overview

The `$mergeObjects` operator in MongoDB aggregation combines multiple objects into a single object. When documents have overlapping field names, later objects in the list override earlier ones - making `$mergeObjects` useful for applying defaults, merging updates, or combining data from multiple sources within a pipeline.

## Basic Syntax

`$mergeObjects` accepts an array of objects or expressions that evaluate to objects:

```javascript
db.products.aggregate([
  {
    $project: {
      merged: {
        $mergeObjects: [
          "$defaults",
          "$overrides"
        ]
      }
    }
  }
])
```

Fields from `$overrides` take precedence over fields from `$defaults` when both have the same key.

## Applying Default Values to Documents

Use `$mergeObjects` to supply default values for missing fields:

```javascript
db.users.aggregate([
  {
    $project: {
      profile: {
        $mergeObjects: [
          {
            theme: "light",
            language: "en",
            notifications: true,
            timezone: "UTC"
          },
          "$preferences"
        ]
      }
    }
  }
])
```

Any field in `$preferences` will override the defaults, and missing fields will fall back to the defaults.

## Merging a Document with a Computed Subset

Add computed fields to a sub-document using `$mergeObjects`:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderInfo: {
        $mergeObjects: [
          "$shipping",
          {
            estimatedArrival: {
              $dateAdd: {
                startDate: "$shipping.dispatchDate",
                unit: "day",
                amount: "$shipping.estimatedDays"
              }
            }
          }
        ]
      }
    }
  }
])
```

## Using $mergeObjects as an Accumulator in $group

When used in `$group`, `$mergeObjects` combines all documents in the group into a single merged object - each field from later documents overrides the same field from earlier ones:

```javascript
db.settings.aggregate([
  {
    $group: {
      _id: "$userId",
      mergedConfig: { $mergeObjects: "$config" }
    }
  }
])
```

This is useful when you have multiple config documents per user and want to merge them all into one, with later entries taking precedence.

## Reshaping Documents with $mergeObjects and $$ROOT

Promote nested fields to the top level while keeping the rest of the document:

```javascript
db.employees.aggregate([
  {
    $project: {
      employee: {
        $mergeObjects: [
          "$$ROOT",
          {
            fullName: {
              $concat: ["$firstName", " ", "$lastName"]
            },
            displayEmail: { $toLower: "$email" }
          }
        ]
      }
    }
  },
  { $replaceRoot: { newRoot: "$employee" } }
])
```

## Conditional Merging

Merge an additional object only when a condition is met:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderData: {
        $mergeObjects: [
          "$$ROOT",
          {
            $cond: {
              if: { $eq: ["$isPriority", true] },
              then: { priorityTag: "RUSH", slaHours: 4 },
              else: {}
            }
          }
        ]
      }
    }
  }
])
```

## Merging Lookup Results

After a `$lookup`, merge the looked-up fields directly into the base document:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerArr"
    }
  },
  {
    $project: {
      enrichedOrder: {
        $mergeObjects: [
          "$$ROOT",
          { $arrayElemAt: ["$customerArr", 0] }
        ]
      }
    }
  },
  { $replaceRoot: { newRoot: "$enrichedOrder" } }
])
```

## Summary

`$mergeObjects` is a versatile operator for combining, extending, and defaulting objects in MongoDB aggregation. As a `$project` expression it merges at the document level, and as a `$group` accumulator it combines multiple documents per group. Its override semantics (later keys win) make it ideal for applying configuration overrides, providing defaults, and enriching documents with computed fields - all within the pipeline.
