# How to Use $and to Combine Multiple Conditions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $and, Logical Operator, Filter

Description: Learn how to use MongoDB's $and operator to combine multiple query conditions, when it is implicit, and when you must use the explicit form.

---

## What Is $and

The `$and` operator joins multiple query conditions with a logical AND. A document matches only if all conditions are satisfied simultaneously.

Syntax:

```javascript
{ $and: [condition1, condition2, condition3] }
```

## Implicit $and with a Single Document

When you specify multiple conditions in a single query document, MongoDB applies an implicit AND. The following two queries are equivalent:

```javascript
// Implicit AND
db.orders.find({ status: "active", total: { $gt: 100 } })

// Explicit $and
db.orders.find({
  $and: [
    { status: "active" },
    { total: { $gt: 100 } }
  ]
})
```

The implicit form is shorter and preferred for simple conditions.

## When You Must Use Explicit $and

You need explicit `$and` when the same field appears multiple times in the filter, because a JavaScript object cannot have duplicate keys:

```javascript
// WRONG: second condition overwrites the first (duplicate key)
db.products.find({ price: { $gt: 10 }, price: { $lt: 100 } })

// CORRECT: use explicit $and
db.products.find({
  $and: [
    { price: { $gt: 10 } },
    { price: { $lt: 100 } }
  ]
})

// OR: merge into a single condition object (also works)
db.products.find({ price: { $gt: 10, $lt: 100 } })
```

## Combining $and with $or

Explicit `$and` is also necessary when combining `$and` with `$or` at the same level:

```javascript
db.users.find({
  $and: [
    { $or: [{ role: "admin" }, { role: "editor" }] },
    { $or: [{ status: "active" }, { verified: true }] }
  ]
})
```

Without explicit `$and`, you cannot have two `$or` keys in the same object.

## Practical Example: Complex Filter

```javascript
db.products.find({
  $and: [
    { category: "electronics" },
    { price: { $gte: 50, $lte: 500 } },
    { $or: [{ brand: "TechCo" }, { featured: true }] },
    { "availability.inStock": true }
  ]
})
```

## $and in the Aggregation Pipeline

```javascript
db.orders.aggregate([
  {
    $match: {
      $and: [
        { status: { $in: ["pending", "processing"] } },
        { createdAt: { $gte: new Date("2025-01-01") } }
      ]
    }
  }
])
```

## Index Behavior with $and

MongoDB's query planner can use indexes for individual conditions within `$and`. It evaluates which index reduces the result set the most and applies it first, then filters the remaining candidates.

```javascript
// With indexes on both status and createdAt, MongoDB picks the most selective
db.orders.find({
  $and: [
    { status: "pending" },
    { createdAt: { $gte: lastWeek } }
  ]
})
```

## Common Mistakes

- Using implicit AND with the same field twice - the second condition silently overwrites the first in JavaScript objects.
- Nesting `$and` inside `$and` unnecessarily - flatten the conditions.
- Confusing `$and` (all must match) with `$or` (any must match).

## Summary

MongoDB's `$and` operator requires all listed conditions to be true for a document to match. For simple multi-field conditions, the implicit AND syntax is preferred. Use explicit `$and` when you need multiple conditions on the same field, or when combining multiple `$or` clauses at the same query level.
