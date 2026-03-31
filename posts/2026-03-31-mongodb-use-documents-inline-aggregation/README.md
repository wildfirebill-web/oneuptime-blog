# How to Use $documents to Generate Inline Documents in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Inline Data

Description: Learn how MongoDB's $documents stage generates a sequence of inline documents within an aggregation pipeline, enabling constant data, lookup tables, and test pipelines without a backing collection.

---

## What Is the $documents Stage?

The `$documents` stage generates a sequence of documents from a literal array expression, injecting them directly into the aggregation pipeline as if they came from a collection. This enables you to create inline lookup tables, generate test data, or combine constant values with `$lookup` and `$unionWith` - all without creating a dedicated collection.

The `$documents` stage must be used in a `db.aggregate()` call rather than a collection-level aggregate.

## Basic Syntax

```javascript
db.aggregate([
  {
    $documents: [
      { <field>: <value>, ... },
      { <field>: <value>, ... }
    ]
  }
])
```

## Example: Generate Constant Documents

```javascript
db.aggregate([
  {
    $documents: [
      { x: 1, label: "one" },
      { x: 2, label: "two" },
      { x: 3, label: "three" }
    ]
  }
])
// Output: three documents exactly as defined
```

## Creating a Lookup Table Inline

```javascript
db.aggregate([
  {
    $documents: [
      { code: "USD", symbol: "$", rate: 1.0 },
      { code: "EUR", symbol: "€", rate: 0.92 },
      { code: "GBP", symbol: "£", rate: 0.79 }
    ]
  },
  {
    $lookup: {
      from: "prices",
      localField: "code",
      foreignField: "currency",
      as: "items"
    }
  }
])
```

This creates an inline currency table and joins it against a prices collection.

## Testing Aggregation Logic Without Real Data

```javascript
db.aggregate([
  {
    $documents: [
      { score: 45, student: "Alice" },
      { score: 78, student: "Bob" },
      { score: 92, student: "Carol" },
      { score: 60, student: "Dave" }
    ]
  },
  {
    $addFields: {
      grade: {
        $switch: {
          branches: [
            { case: { $gte: ["$score", 90] }, then: "A" },
            { case: { $gte: ["$score", 80] }, then: "B" },
            { case: { $gte: ["$score", 70] }, then: "C" },
            { case: { $gte: ["$score", 60] }, then: "D" }
          ],
          default: "F"
        }
      }
    }
  }
])
// Tests grade assignment logic with inline data
```

## Generating a Date Sequence

```javascript
db.aggregate([
  {
    $documents: [
      { month: 1, name: "January" },
      { month: 2, name: "February" },
      { month: 3, name: "March" }
    ]
  },
  {
    $lookup: {
      from: "salesData",
      let: { m: "$month" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: [{ $month: "$saleDate" }, "$$m"] }
          }
        },
        { $group: { _id: null, total: { $sum: "$amount" } } }
      ],
      as: "monthlySales"
    }
  }
])
```

## Combining $documents with $unionWith

```javascript
db.aggregate([
  {
    $documents: [
      { type: "synthetic", value: 100 }
    ]
  },
  {
    $unionWith: {
      coll: "realMetrics",
      pipeline: [{ $project: { type: { $literal: "real" }, value: 1 } }]
    }
  }
])
```

## Using Expressions in $documents

```javascript
db.aggregate([
  {
    $documents: [
      { date: new Date("2024-01-01"), amount: 500 },
      { date: new Date("2024-06-01"), amount: 750 }
    ]
  },
  {
    $addFields: {
      year: { $year: "$date" },
      quarter: { $ceil: { $divide: [{ $month: "$date" }, 3] } }
    }
  }
])
```

## $documents vs Collection Queries

| Feature | $documents | Collection Query |
|---------|------------|-----------------|
| Data source | Inline literal | Persisted collection |
| Requires collection | No | Yes |
| Supports expressions | Yes | Yes |
| Best for | Constants, test data, lookup tables | Real application data |

## Summary

The `$documents` stage creates inline document sequences directly in an aggregation pipeline. It is invaluable for testing pipeline logic without real data, building inline reference tables for `$lookup`, and combining constant values with real collection data. Use it in `db.aggregate()` rather than `db.collection.aggregate()`.
