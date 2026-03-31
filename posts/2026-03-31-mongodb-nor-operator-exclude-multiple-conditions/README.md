# How to Use $nor to Exclude Multiple Conditions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $nor, Logical Operator, Filter

Description: Learn how MongoDB's $nor operator matches documents that fail all listed conditions, and when to use it over $not or $and with $ne operators.

---

## What Is $nor

The `$nor` operator matches documents that do not satisfy any of the conditions in the array. A document matches only if every condition in the `$nor` array is false for that document.

Syntax:

```javascript
{ $nor: [condition1, condition2, condition3] }
```

Think of it as: NOT(condition1) AND NOT(condition2) AND NOT(condition3).

## Basic Example

Find documents that are neither cancelled nor archived:

```javascript
db.orders.find({
  $nor: [
    { status: "cancelled" },
    { status: "archived" }
  ]
})
```

This is equivalent to:

```javascript
db.orders.find({
  status: { $nin: ["cancelled", "archived"] }
})
```

For simple same-field exclusions, `$nin` is more concise. `$nor` shines when conditions span multiple fields.

## Multi-Field Exclusion

Exclude documents that match any of several different conditions:

```javascript
db.users.find({
  $nor: [
    { role: "bot" },
    { status: "suspended" },
    { emailVerified: false }
  ]
})
```

This returns users who are not bots, not suspended, and have verified emails. You cannot express this as cleanly with `$nin` since the conditions are on different fields.

## $nor and Missing Fields

Like `$not` and `$ne`, `$nor` matches documents where fields do not exist:

```javascript
// Also matches documents where "priority" field is missing entirely
db.tasks.find({
  $nor: [
    { priority: "low" },
    { priority: "medium" }
  ]
})
```

Combine with `$exists` to limit results to documents that have all relevant fields.

## Combining $nor with Other Operators

```javascript
db.products.find({
  $nor: [
    { price: { $gt: 1000 } },
    { category: "luxury" },
    { discontinued: true }
  ]
})
```

Matches products with price at most $1000, not in the luxury category, and not discontinued.

## $nor vs $and with $ne

`$nor` is a shorthand for `$and` with `$ne` on each condition:

```javascript
// Using $nor
db.orders.find({
  $nor: [{ status: "cancelled" }, { status: "refunded" }]
})

// Equivalent using $and + $ne
db.orders.find({
  $and: [
    { status: { $ne: "cancelled" } },
    { status: { $ne: "refunded" } }
  ]
})
```

`$nor` is more readable when you have many conditions to exclude.

## $nor in Aggregation

```javascript
db.events.aggregate([
  {
    $match: {
      $nor: [
        { type: "test" },
        { internal: true }
      ]
    }
  },
  { $count: "publicEvents" }
])
```

## Performance Considerations

`$nor` requires MongoDB to verify all conditions are false for each document. Ensure that the fields used in `$nor` conditions are indexed. If any condition's field lacks an index, the query degrades to a collection scan.

## Common Mistakes

- Using `$nor` when `$nin` would be more appropriate for same-field exclusions.
- Forgetting that `$nor` also matches documents where the condition fields are absent.
- Confusing `$nor` with `$not` - `$not` wraps a single field expression, `$nor` takes an array of full query conditions.

## Summary

`$nor` matches documents that fail every condition in its array - the inverse of `$or`. It is most useful for multi-field exclusion filters. For single-field exclusion of multiple values, `$nin` is simpler. Be aware that like other negation operators, `$nor` matches documents with missing fields unless you explicitly add `$exists` checks.
