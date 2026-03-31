# How to Use $not to Negate Query Conditions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $not, Logical Operator, Filter

Description: Learn how MongoDB's $not operator inverts a query expression, including its syntax restrictions and how it differs from $ne and $nor.

---

## What Is $not

The `$not` operator inverts the effect of a query expression. A document matches if the condition inside `$not` is false or if the field does not exist.

Syntax:

```javascript
{ field: { $not: { operator-expression } } }
```

Note that `$not` wraps an operator expression applied to a specific field, unlike `$nor` and `$and` which take an array of full query conditions.

## Basic Example

Find products where price is NOT greater than 100:

```javascript
db.products.find({ price: { $not: { $gt: 100 } } })
```

This is equivalent to `{ price: { $lte: 100 } }` but also matches documents where `price` is absent.

## $not with Regular Expressions

One of the most practical uses of `$not` is inverting a regex:

```javascript
// Find users whose email does NOT end with @example.com
db.users.find({ email: { $not: /\@example\.com$/ } })
```

Using `$not` with a regex is the only way to express "field does not match pattern" in a query filter.

## $not Includes Missing Fields

Like `$ne` and `$nin`, `$not` matches documents where the field is absent:

```javascript
// Matches: priority is not "low" AND documents where priority is missing
db.tasks.find({ priority: { $not: { $eq: "low" } } })
```

To exclude documents with missing fields, combine with `$exists`:

```javascript
db.tasks.find({
  priority: { $exists: true },
  priority: { $not: { $eq: "low" } }
})
```

Note: you cannot have the same key twice in a JS object. Use `$and`:

```javascript
db.tasks.find({
  $and: [
    { priority: { $exists: true } },
    { priority: { $not: { $eq: "low" } } }
  ]
})
```

## $not vs $ne

`$not: { $eq: value }` and `$ne: value` produce the same results:

```javascript
// These are equivalent
db.users.find({ status: { $ne: "inactive" } })
db.users.find({ status: { $not: { $eq: "inactive" } } })
```

Prefer `$ne` for simple inequality checks. Use `$not` for inverting complex expressions like ranges or regex patterns.

## Inverting a Range

```javascript
// Price is NOT between 50 and 100 (i.e., below 50 or above 100)
db.products.find({
  price: { $not: { $gte: 50, $lte: 100 } }
})
```

## $not in Aggregation

In the aggregation pipeline, the equivalent operator is `$not` as an expression returning boolean:

```javascript
db.orders.aggregate([
  {
    $project: {
      isNotPending: { $not: [{ $eq: ["$status", "pending"] }] }
    }
  }
])
```

## Common Mistakes

- Trying to use `$not` at the top level like `$and` or `$or` - `$not` must wrap an operator expression on a specific field.
- Forgetting that `$not` matches documents where the field is missing.
- Using `$not: value` without an operator expression inside - this is a syntax error.

## Summary

`$not` negates an operator expression applied to a specific field, matching documents where the expression is false or the field is absent. Use it primarily for inverting regex patterns and complex range conditions. For simple inequality, `$ne` is more direct and readable. Remember to combine with `$exists` if you want to exclude documents where the field is missing.
