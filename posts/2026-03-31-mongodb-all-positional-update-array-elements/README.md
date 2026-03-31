# How to Use $[] to Update All Elements in an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Array

Description: Learn how MongoDB's $[] all-positional operator updates every element in an array field, and when to use it over the $ positional operator.

---

The `$[]` operator, introduced in MongoDB 3.6, is the all-positional operator. Unlike the `$` operator which updates only the first matching element, `$[]` updates **every** element in an array field regardless of its value. It is useful when you want to apply the same change to all array elements without filtering.

## Syntax

```javascript
db.collection.updateOne(
  { filter },
  { $set: { "arrayField.$[].subField": newValue } }
)
```

The `$[]` placeholder represents all elements in the array at that position.

## Updating All Elements in a Simple Array

Set all scores in a results array to zero:

```javascript
db.quizzes.updateOne(
  { _id: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1") },
  { $set: { "scores.$[]": 0 } }
)
```

Before: `{ scores: [85, 72, 90] }`
After: `{ scores: [0, 0, 0] }`

## Updating a Sub-Field in All Array Objects

A student document has a `grades` array of objects. Mark all grades as reviewed:

```javascript
db.students.updateMany(
  { courseId: "CS101" },
  { $set: { "grades.$[].reviewed": true } }
)
```

This applies to every element in the `grades` array for each matched document.

## Incrementing a Field Across All Elements

Add a bonus to every player's score in a game document:

```javascript
db.games.updateOne(
  { _id: "game_12" },
  { $inc: { "players.$[].score": 5 } }
)
```

## Combining $[] with $push on Nested Arrays

Push a tag into every product's `tags` sub-array:

```javascript
db.catalogs.updateMany(
  { category: "electronics" },
  { $push: { "products.$[].tags": "sale" } }
)
```

## $[] vs. $ Positional Operator

| Operator | Scope | Filter Required |
|---|---|---|
| `$` | First matching element | Yes, in query |
| `$[]` | All elements | No |
| `$[identifier]` | Filtered elements | Yes, in arrayFilters |

Use `$[]` when you want to blanket-apply a change and do not need to target specific elements.

## Updating Nested Arrays with $[]

`$[]` can be chained for nested arrays. Flatten all nested score lists:

```javascript
db.reports.updateOne(
  { _id: "report_5" },
  { $set: { "sections.$[].scores.$[]": 100 } }
)
```

This sets every score inside every section's `scores` array to 100.

## Performance Consideration

`$[]` iterates over every element in the array. For large arrays with many elements, this can increase write cost. If you only need to update specific elements, `$[identifier]` with `arrayFilters` is more efficient.

## Summary

`$[]` updates all elements in an array field without needing a filter condition. It simplifies bulk resets, default value assignments, and cross-element modifications compared to iterating manually. For selective updates, use `$[identifier]` with `arrayFilters`; for the first-match scenario, use the `$` positional operator.
