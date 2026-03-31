# How to Use $pull to Remove Matching Elements from an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Operator, Update Operator, Document

Description: Learn how MongoDB's $pull operator removes all elements from an array that match a specified value or condition, covering scalar values, objects, and query operators.

---

## What Is the $pull Operator?

The `$pull` operator removes all array elements that match a given condition from a document field. Unlike `$pop` which removes by position, `$pull` removes by value or query expression, making it the right tool when you know what to remove but not where it sits in the array.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  { $pull: { <arrayField>: <valueOrCondition> } }
)
```

## Example: Removing a Specific Value

```javascript
// Document: { _id: 1, scores: [85, 90, 95, 90] }

db.students.updateOne(
  { _id: 1 },
  { $pull: { scores: 90 } }
)
// scores becomes: [85, 95]
// Both occurrences of 90 are removed
```

## Removing a Specific Tag

```javascript
db.articles.updateOne(
  { _id: "article-1" },
  { $pull: { tags: "deprecated" } }
)
```

## Removing Elements by Condition

Use query operators to remove elements that match a condition.

```javascript
// Remove all scores below 60
db.students.updateOne(
  { _id: 1 },
  { $pull: { scores: { $lt: 60 } } }
)
```

## Removing Objects from Arrays

When pulling objects, you can match on one or more fields.

```javascript
// Document: { _id: "cart-1", items: [{ id: "A", qty: 2 }, { id: "B", qty: 1 }] }

db.carts.updateOne(
  { _id: "cart-1" },
  { $pull: { items: { id: "A" } } }
)
// Removes all items where id equals "A"
```

## Removing Objects with Complex Conditions

```javascript
db.logs.updateMany(
  {},
  { $pull: { events: { level: "debug", ts: { $lt: cutoffDate } } } }
)
```

## Pulling from Multiple Arrays in One Operation

```javascript
db.users.updateOne(
  { _id: userId },
  {
    $pull: {
      blockedUsers: targetId,
      pendingRequests: targetId
    }
  }
)
```

## Bulk Removal Across Documents

```javascript
db.products.updateMany(
  { category: "clothing" },
  { $pull: { colors: "beige" } }
)
```

## $pull vs $pullAll

- `$pull` removes elements matching a query condition (supports operators like `$gt`, `$lt`, `$in`)
- `$pullAll` removes all elements equal to any value in a provided list (exact equality only)

```javascript
// $pullAll example - removes exact matches
db.items.updateOne(
  { _id: 1 },
  { $pullAll: { sizes: ["XS", "XXL"] } }
)
```

Use `$pull` when you need flexible conditions; use `$pullAll` when removing a known list of exact values.

## Summary

The `$pull` operator is the most flexible way to remove array elements in MongoDB. It supports scalar equality, query expressions, and object field matching, removing all matching occurrences in a single atomic operation. For removing an exact set of values, `$pullAll` is a simpler alternative.
