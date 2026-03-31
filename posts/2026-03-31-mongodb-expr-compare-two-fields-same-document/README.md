# How to Use $expr to Compare Two Fields in the Same Document in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $expr, Aggregation Expression, Query, Field Comparison

Description: Learn how to use MongoDB's $expr operator to compare two fields within the same document or use aggregation expressions inside find() queries.

---

## What Is $expr

The `$expr` operator lets you use aggregation expressions inside a regular query filter. Its most important use is comparing two fields within the same document - something that standard query operators cannot do.

Syntax:

```javascript
{ $expr: { aggregation-expression } }
```

## Comparing Two Fields in the Same Document

Without `$expr`, you cannot write a filter like "find orders where discount is greater than tax". With `$expr`:

```javascript
// Find orders where the discount is greater than the tax amount
db.orders.find({
  $expr: { $gt: ["$discount", "$tax"] }
})
```

The `$` prefix before field names tells MongoDB to use the field value, not a literal string.

## Common Field Comparison Expressions

```javascript
// Documents where endDate is after startDate
db.events.find({
  $expr: { $gt: ["$endDate", "$startDate"] }
})

// Documents where stock is less than minStock threshold
db.inventory.find({
  $expr: { $lt: ["$currentStock", "$reorderPoint"] }
})

// Documents where actual spend exceeds the budget
db.projects.find({
  $expr: { $gt: ["$actualCost", "$budget"] }
})
```

## Using Arithmetic in $expr

You can embed arithmetic operators to compare derived values:

```javascript
// Find orders where shipping cost is more than 10% of the total
db.orders.find({
  $expr: {
    $gt: [
      "$shippingCost",
      { $multiply: ["$total", 0.1] }
    ]
  }
})
```

## Combining $expr with Standard Query Operators

```javascript
db.orders.find({
  status: "shipped",
  $expr: { $gt: ["$deliveredAt", "$estimatedDelivery"] }
})
```

This finds shipped orders that arrived late.

## $expr with $cond for Conditional Logic

```javascript
// Flag documents where the effective price differs from base price
db.products.find({
  $expr: {
    $ne: [
      "$effectivePrice",
      {
        $cond: {
          if: { $gt: ["$discount", 0] },
          then: { $subtract: ["$basePrice", "$discount"] },
          else: "$basePrice"
        }
      }
    ]
  }
})
```

## $expr in the Aggregation Pipeline

`$expr` is also valid in `$match` stages:

```javascript
db.subscriptions.aggregate([
  {
    $match: {
      $expr: { $lt: ["$usedStorage", "$storageLimit"] }
    }
  },
  {
    $project: {
      user: 1,
      percentUsed: {
        $multiply: [{ $divide: ["$usedStorage", "$storageLimit"] }, 100]
      }
    }
  }
])
```

## Index Behavior with $expr

`$expr` queries generally cannot use standard indexes because the comparison involves computed values. For frequently executed `$expr` queries:

- Consider adding computed fields to documents and indexing them.
- Use the aggregation pipeline to compute and cache values.

```javascript
// Pre-compute and store the field for fast indexing
await db.collection("orders").updateMany(
  {},
  [{ $set: { isLate: { $gt: ["$deliveredAt", "$estimatedDelivery"] } } }]
);
await db.collection("orders").createIndex({ isLate: 1 });
```

## Common Mistakes

- Forgetting the `$` prefix on field names inside `$expr` - without it, MongoDB treats the value as a literal string.
- Expecting index support for `$expr` comparisons - they typically require a full scan.
- Using `$expr` where a standard comparison operator would work - `$expr` adds complexity without benefit for simple scalar comparisons.

## Summary

`$expr` enables aggregation expressions within query filters, making it possible to compare two fields in the same document or apply arithmetic before comparing. Use it when you need cross-field comparisons or computed value checks. For performance, consider materializing frequently queried expressions as indexed fields on the document.
