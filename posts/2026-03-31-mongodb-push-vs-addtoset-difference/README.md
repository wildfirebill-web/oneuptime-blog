# What Is the Difference Between $push and $addToSet in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Push, AddToSet, Array, Update Operator

Description: In MongoDB, $push appends a value to an array regardless of duplicates, while $addToSet appends only if the value is not already present, acting like a set.

---

## Overview

Both `$push` and `$addToSet` are MongoDB update operators that add elements to an array field. The key difference is their behavior with duplicates: `$push` always appends the value, potentially creating duplicates, while `$addToSet` only appends the value if it does not already exist in the array. Choosing between them depends on whether your array should behave like a list (ordered, allows duplicates) or a set (unordered, unique values only).

## $push: Append Always

`$push` adds the specified value to the end of an array field. If the value already exists in the array, it still gets added again.

```javascript
// Starting document: { _id: 1, tags: ["mongodb"] }

// First push
db.posts.updateOne(
  { _id: 1 },
  { $push: { tags: "database" } }
)
// Result: { tags: ["mongodb", "database"] }

// Push same value again
db.posts.updateOne(
  { _id: 1 },
  { $push: { tags: "database" } }
)
// Result: { tags: ["mongodb", "database", "database"] }
// Duplicate!
```

## $addToSet: Append Only If Unique

`$addToSet` adds the value only if it is not already present in the array. It guarantees uniqueness.

```javascript
// Starting document: { _id: 1, tags: ["mongodb"] }

// First addToSet
db.posts.updateOne(
  { _id: 1 },
  { $addToSet: { tags: "database" } }
)
// Result: { tags: ["mongodb", "database"] }

// Attempt to add duplicate
db.posts.updateOne(
  { _id: 1 },
  { $addToSet: { tags: "database" } }
)
// Result: { tags: ["mongodb", "database"] }
// No duplicate added
```

## $each Modifier

Both `$push` and `$addToSet` support the `$each` modifier to add multiple values at once:

```javascript
// Push multiple values (allows duplicates)
db.posts.updateOne(
  { _id: 1 },
  { $push: { comments: { $each: ["comment1", "comment2"] } } }
)

// Add multiple unique values
db.posts.updateOne(
  { _id: 1 },
  { $addToSet: { tags: { $each: ["nosql", "cloud", "mongodb"] } } }
)
// "mongodb" won't be added if already present; others will be
```

## $push with $sort and $slice

`$push` supports additional modifiers for sorted, capped arrays:

```javascript
// Keep only the top 5 highest scores in a sorted array
db.leaderboard.updateOne(
  { userId: "player1" },
  {
    $push: {
      scores: {
        $each: [{ score: 9500, level: 10 }],
        $sort: { score: -1 },
        $slice: 5
      }
    }
  }
)
```

`$addToSet` does not support `$sort` or `$slice`.

## Object Equality in $addToSet

`$addToSet` considers objects equal only if they have exactly the same fields with the same values in the same order. If field order differs, it treats them as different objects:

```javascript
// These are treated as DIFFERENT by $addToSet due to field order
{ a: 1, b: 2 }  vs  { b: 2, a: 1 }

// This can lead to unexpected duplicates - be careful with objects
```

## When to Use Each

| Scenario | Use |
|---|---|
| Activity log, comment list | `$push` - order and duplicates may be valid |
| Tag list, permission set | `$addToSet` - uniqueness required |
| Top-N list with sorting | `$push` with `$sort` and `$slice` |
| Collect unique user IDs | `$addToSet` |
| Queue entries | `$push` |

## Creating a Set from Scratch

If the array field does not exist, both operators create it:

```javascript
// Creates { followers: ["user123"] }
db.users.updateOne(
  { _id: "author1" },
  { $addToSet: { followers: "user123" } },
  { upsert: true }
)
```

## Summary

Use `$push` when you want to append elements to an array and duplicates are acceptable or desired. Use `$addToSet` when the array must contain only unique values. Both operators support `$each` for batch additions, but only `$push` supports `$sort` and `$slice` for maintaining sorted, size-limited arrays.
