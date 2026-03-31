# How to Debug Aggregation Pipelines Stage by Stage in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Debugging, Pipeline, Performance

Description: Learn how to debug MongoDB aggregation pipelines stage by stage to identify incorrect results, unexpected document shapes, and performance bottlenecks.

---

## Why Stage-by-Stage Debugging Is Essential

Complex aggregation pipelines can produce incorrect results for reasons that are not obvious when viewing only the final output. A wrong `$group` key, an unexpected `$unwind` expansion, or a `$lookup` that matches nothing can silently corrupt results. Debugging each stage in isolation reveals exactly where the pipeline goes wrong.

## Running a Partial Pipeline

The most direct debugging technique is to progressively add stages and inspect the output after each one. In the MongoDB shell or Compass, run the pipeline with only the first stage, then two stages, and so on.

```javascript
// Step 1: Check raw match results
db.orders.aggregate([
  { $match: { status: "pending", region: "US" } }
]).pretty();

// Step 2: Add projection and verify document shape
db.orders.aggregate([
  { $match: { status: "pending", region: "US" } },
  { $project: { customerId: 1, total: 1 } }
]).pretty();

// Step 3: Add group and verify counts
db.orders.aggregate([
  { $match: { status: "pending", region: "US" } },
  { $project: { customerId: 1, total: 1 } },
  { $group: { _id: "$customerId", orderCount: { $sum: 1 }, totalSpend: { $sum: "$total" } } }
]).pretty();
```

This isolates the exact stage where results diverge from expectations.

## Using $sample to Work with Subsets

On large collections, running debug pipelines against the full dataset is slow. Use `$sample` to work with a representative subset.

```javascript
db.events.aggregate([
  { $sample: { size: 100 } },   // Random 100 documents
  { $match: { type: "error" } },
  { $group: { _id: "$service", count: { $sum: 1 } } }
]);
```

This gives you fast feedback cycles while debugging logic.

## Inserting $project to Inspect Document Shape

After stages like `$lookup` or `$unwind`, use a temporary `$project` to print the current document shape and verify that fields are named and nested as expected.

```javascript
db.invoices.aggregate([
  { $match: { paid: false } },
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerData"
  }},
  // Debug: check what customerData looks like
  { $project: { customerId: 1, paid: 1, customerData: 1 } }
]);
```

If `customerData` is an empty array, the join is not matching. If it has unexpected fields, adjust your projection.

## Using explain() to Detect Stage Issues

The `explain()` command reveals how many documents each stage processes and whether indexes are being used.

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "pending" } },
  { $group: { _id: "$customerId", count: { $sum: 1 } } }
]);
```

Look at `nReturned` and `totalDocsExamined` to confirm expected document flow between stages.

## Checking for null and Missing Fields

A common source of incorrect aggregation results is documents with missing or null fields. Use `$match` to check for these during debugging.

```javascript
// Find documents where the field is missing
db.orders.aggregate([
  { $match: { customerId: { $exists: false } } },
  { $count: "missingCustomerId" }
]);

// Find documents with null values
db.orders.aggregate([
  { $match: { total: null } },
  { $limit: 5 }
]);
```

## Summary

Debugging MongoDB aggregation pipelines effectively requires running the pipeline incrementally, one stage at a time, and verifying document counts and shapes at each step. Use `$sample` to test on subsets, insert temporary `$project` stages to inspect intermediate document shapes, and use `explain("executionStats")` to identify stages with unexpectedly high document counts. Checking for missing or null field values before grouping or sorting prevents silent errors in final results.
