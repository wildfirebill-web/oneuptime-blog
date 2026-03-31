# How to Combine $and and $or for Complex Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $and, $or, Logical Operator

Description: Learn how to combine MongoDB's $and and $or operators to express complex multi-condition queries with practical examples and index strategies.

---

## Why Combine $and and $or

Real-world queries often need conditions like "match this AND (one of these OR that)". MongoDB's `$and` and `$or` operators can be nested to express arbitrarily complex boolean logic.

## Basic Combined Query

Find active products that are either featured or on sale:

```javascript
db.products.find({
  status: "active",
  $or: [
    { featured: true },
    { discount: { $gt: 0 } }
  ]
})
```

The implicit `$and` combines `status: "active"` with the `$or` group. A product must be active AND (featured OR discounted).

## Multiple $or Groups Require Explicit $and

A plain JavaScript object cannot have two `$or` keys. To combine multiple OR groups, use explicit `$and`:

```javascript
// WRONG - second $or silently overwrites first
db.users.find({
  $or: [{ role: "admin" }, { role: "editor" }],
  $or: [{ status: "active" }, { verified: true }]
})

// CORRECT - explicit $and wrapping two $or groups
db.users.find({
  $and: [
    { $or: [{ role: "admin" }, { role: "editor" }] },
    { $or: [{ status: "active" }, { verified: true }] }
  ]
})
```

This matches users who are (admin or editor) AND (active or verified).

## Three-Level Nesting Example

Find VIP users OR users from US with premium plan AND who have logged in recently:

```javascript
const lastMonth = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);

db.users.find({
  lastLogin: { $gte: lastMonth },
  $or: [
    { tier: "vip" },
    {
      $and: [
        { country: "US" },
        { plan: "premium" }
      ]
    }
  ]
})
```

## Practical Example: Product Search

Search for products in the right price range, from specific brands or categories, that are in stock:

```javascript
db.products.find({
  $and: [
    { price: { $gte: 50, $lte: 500 } },
    { inStock: true },
    {
      $or: [
        { brand: { $in: ["BrandA", "BrandB"] } },
        { category: "featured" }
      ]
    }
  ]
})
```

## Index Strategy for Combined Queries

MongoDB's query planner evaluates each branch to pick the best index. For combined `$and`/`$or` queries:

1. Index the most selective field first (typically the field with the fewest matching documents).
2. For each `$or` branch, ensure there is a supporting index.

```javascript
await db.collection("orders").createIndex({ status: 1, createdAt: -1 });
await db.collection("orders").createIndex({ assignedTo: 1 });

db.orders.find({
  status: "pending",
  $or: [
    { createdAt: { $lt: urgentThreshold } },
    { assignedTo: currentUser }
  ]
})
```

## Verifying with explain()

Always validate complex query plans:

```javascript
db.products.find({
  $and: [
    { status: "active" },
    { $or: [{ featured: true }, { discount: { $gt: 0 } }] }
  ]
}).explain("executionStats")
```

Look for `IXSCAN` stages and low `totalDocsExamined` relative to `nReturned`.

## Common Mistakes

- Using two `$or` keys in one object - always use explicit `$and` to wrap multiple `$or` groups.
- Deeply nesting conditions when a flatter structure works.
- Not checking the query plan with `explain()` after adding complex logic.

## Summary

Combining `$and` and `$or` lets you express complex boolean logic in MongoDB queries. Use the implicit AND for simple multi-field conditions, and explicit `$and` when you need to combine multiple `$or` groups at the same level. Always verify complex queries with `explain()` and ensure each `$or` branch has index support.
