# How to Use $project to Reduce Document Size Early in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline, Performance, Projection

Description: Learn how to use $project early in MongoDB aggregation pipelines to reduce document size, lower memory usage, and speed up downstream stages.

---

## Why Document Size Matters in Aggregation

Every document that passes through a MongoDB aggregation pipeline occupies memory. If your documents contain large arrays, embedded subdocuments, or many fields that are irrelevant to your query goal, carrying them through every stage wastes resources and slows execution.

Using `$project` early in the pipeline - right after `$match` - is a proven technique for shrinking document payloads and improving overall pipeline throughput.

## Basic Field Inclusion and Exclusion

`$project` can either include only the fields you need (inclusion projection) or exclude specific fields you do not need (exclusion projection).

```javascript
// Inclusion: keep only relevant fields
db.users.aggregate([
  { $match: { active: true } },
  { $project: { name: 1, email: 1, plan: 1 } },
  { $group: { _id: "$plan", count: { $sum: 1 } } }
]);

// Exclusion: drop large or sensitive fields
db.events.aggregate([
  { $match: { type: "click" } },
  { $project: { rawPayload: 0, debugInfo: 0 } },
  { $group: { _id: "$sessionId", clicks: { $sum: 1 } } }
]);
```

Do not mix inclusion and exclusion in the same `$project` (except for `_id`).

## Dropping Nested Fields to Shrink Documents

Documents with deeply nested subdocuments or large embedded arrays are common in MongoDB. Use dot notation in `$project` to exclude specific nested fields.

```javascript
db.orders.aggregate([
  { $match: { status: "fulfilled" } },
  {
    $project: {
      customerId: 1,
      total: 1,
      "address.city": 1,
      "address.country": 1
      // Excludes address.street, address.zip, etc.
    }
  },
  { $group: { _id: { city: "$address.city", country: "$address.country" }, revenue: { $sum: "$total" } } }
]);
```

## Combining $project with Computed Fields

You can compute new fields and discard originals in the same stage, avoiding a separate `$addFields` pass.

```javascript
db.sales.aggregate([
  { $match: { region: "EU" } },
  {
    $project: {
      region: 1,
      month: { $month: "$saleDate" },
      revenue: { $multiply: ["$quantity", "$unitPrice"] }
      // Original saleDate, quantity, unitPrice dropped implicitly in inclusion mode
    }
  },
  { $group: { _id: { region: "$region", month: "$month" }, totalRevenue: { $sum: "$revenue" } } }
]);
```

## Using $project Before $lookup

Trimming documents before a `$lookup` stage is especially valuable. The fewer fields a document carries, the less data MongoDB needs to manage during the join operation.

```javascript
db.invoices.aggregate([
  { $match: { paid: false } },
  { $project: { customerId: 1, amount: 1, dueDate: 1 } },
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
  }},
  { $project: { "customer.name": 1, "customer.email": 1, amount: 1, dueDate: 1 } }
]);
```

## Verifying Impact with explain()

Use the `explain()` method with `executionStats` to compare document sizes and counts between stages before and after adding `$project`.

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { year: 2025 } },
  { $project: { customerId: 1, total: 1 } },
  { $group: { _id: "$customerId", spend: { $sum: "$total" } } }
]);
```

Review the `nReturned` and examine stage-level statistics to confirm fewer bytes are flowing through.

## Summary

Placing `$project` immediately after `$match` in MongoDB aggregation pipelines is a simple and effective way to reduce the memory and CPU cost of every subsequent stage. By keeping only the fields needed for the final result and dropping large or unused fields early, you shrink the per-document payload that travels through the pipeline. This approach is especially impactful before `$lookup`, `$unwind`, or `$group` stages, which can amplify document counts and sizes significantly.
