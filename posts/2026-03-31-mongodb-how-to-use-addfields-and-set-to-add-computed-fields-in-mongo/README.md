# How to Use $addFields and $set to Add Computed Fields in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $addFields, $set, Pipeline Stages, NoSQL

Description: Learn how to use MongoDB's $addFields and $set aggregation stages to append computed fields to documents without removing existing fields.

---

## What Are $addFields and $set?

The `$addFields` stage adds new fields to documents passing through the pipeline. Existing fields are preserved - unlike `$project`, which requires you to explicitly include every field you want to keep. `$set` is an alias for `$addFields` introduced in MongoDB 4.2.

```javascript
{ $addFields: { newField: expression } }
// or equivalently:
{ $set: { newField: expression } }
```

## When to Use $addFields vs $project

- Use `$addFields` / `$set` when you want to keep all existing fields and just add new ones.
- Use `$project` when you want precise control over which fields are included.

```javascript
// $project: must list every field you want to keep
{ $project: { name: 1, price: 1, discountedPrice: { $multiply: ["$price", 0.9] } } }

// $addFields: keeps everything and adds the new field
{ $addFields: { discountedPrice: { $multiply: ["$price", 0.9] } } }
```

## Basic Example

Add a computed `fullName` field to user documents:

```javascript
db.users.aggregate([
  {
    $addFields: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] }
    }
  }
])
```

## Adding Multiple Fields

```javascript
db.orders.aggregate([
  {
    $addFields: {
      subtotal: { $multiply: ["$price", "$quantity"] },
      tax: { $multiply: ["$price", "$quantity", 0.08] },
      totalWithTax: {
        $multiply: ["$price", "$quantity", 1.08]
      }
    }
  }
])
```

## Overriding Existing Fields

`$addFields` can override existing field values:

```javascript
db.products.aggregate([
  {
    $addFields: {
      // Override price with a rounded version
      price: { $round: ["$price", 2] },
      // Add a new field
      priceCategory: {
        $cond: [{ $gt: ["$price", 100] }, "premium", "standard"]
      }
    }
  }
])
```

## Adding Fields to Nested Documents

Use dot notation or nested object syntax:

```javascript
db.users.aggregate([
  {
    $addFields: {
      "stats.accountAgeDays": {
        $dateDiff: {
          startDate: "$createdAt",
          endDate: "$$NOW",
          unit: "day"
        }
      }
    }
  }
])
```

## Chaining Multiple $addFields Stages

Each `$addFields` stage can reference fields added by previous stages:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      subtotal: { $multiply: ["$price", "$quantity"] }
    }
  },
  {
    $addFields: {
      // Now we can reference $subtotal from the previous stage
      tax: { $multiply: ["$subtotal", 0.08] },
      total: { $multiply: ["$subtotal", 1.08] }
    }
  }
])
```

## Practical Use Case - Enriching Search Results

Add relevance scores and formatted display fields:

```javascript
db.articles.aggregate([
  { $match: { status: "published" } },
  {
    $addFields: {
      ageInDays: {
        $dateDiff: {
          startDate: "$publishedAt",
          endDate: "$$NOW",
          unit: "day"
        }
      },
      readTimeMinutes: {
        $ceil: { $divide: ["$wordCount", 200] }
      },
      isRecent: {
        $lt: [
          { $dateDiff: {
            startDate: "$publishedAt",
            endDate: "$$NOW",
            unit: "day"
          }},
          7
        ]
      }
    }
  }
])
```

## Using $set in Update Pipeline

`$set` can also be used in pipeline-style updates (not just aggregation):

```javascript
db.users.updateMany(
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

## Removing Fields with $unset in the Same Pipeline

Combine with `$unset` to add and remove fields in one pipeline:

```javascript
db.users.aggregate([
  {
    $addFields: {
      displayName: { $concat: ["$firstName", " ", "$lastName"] }
    }
  },
  {
    $unset: ["firstName", "lastName", "password"]
  }
])
```

## Summary

`$addFields` and its alias `$set` make it easy to enrich documents with computed fields in aggregation pipelines without the verbose field-by-field inclusion required by `$project`. They are ideal for adding derived metrics, reformatting data, and enriching documents before downstream stages like `$group` or `$merge`.
