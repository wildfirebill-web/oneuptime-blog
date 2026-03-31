# How to Use $slice to Limit Array Elements in MongoDB Projections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $slice, Projection, Array, Query

Description: Learn how to use MongoDB's $slice projection operator to return only a subset of array elements, including first N, last N, and range slicing.

---

## What Is $slice

The `$slice` projection operator limits the number of elements returned from an array field. It is used in the projection (second argument to `find()`), not in the filter. It is useful for returning recent items, previewing large arrays, or paginating array contents.

## Basic Syntax

```javascript
// Return first N elements
db.collection.find(filter, { arrayField: { $slice: N } })

// Return last N elements (negative N)
db.collection.find(filter, { arrayField: { $slice: -N } })

// Return N elements starting at offset [skip, limit]
db.collection.find(filter, { arrayField: { $slice: [skip, limit] } })
```

## Return First N Elements

```javascript
// Return only the first 3 comments on each post
db.posts.find(
  { status: "published" },
  { title: 1, comments: { $slice: 3 } }
)
```

## Return Last N Elements

```javascript
// Return the last 5 activity log entries per user
db.users.find(
  {},
  { username: 1, activityLog: { $slice: -5 } }
)
```

Negative values count from the end of the array.

## Return a Range (Skip + Limit)

```javascript
// Skip first 10 comments, return next 5 (pagination within array)
db.posts.find(
  { _id: postId },
  { comments: { $slice: [10, 5] } }
)
```

## Combining $slice with Other Projections

```javascript
// Return title, author, and first 3 tags only
db.articles.find(
  { category: "tech" },
  {
    title: 1,
    author: 1,
    tags: { $slice: 3 }
  }
)
```

Note: you can combine `$slice` with inclusion projections, but you cannot mix inclusion and exclusion (except for `_id`).

## Practical Use Case: Preview the Latest Messages

```javascript
// Show the 10 most recent messages in each conversation
const conversations = await db.collection("conversations").find(
  { participants: userId },
  {
    title: 1,
    messages: { $slice: -10 },
    updatedAt: 1
  }
).sort({ updatedAt: -1 }).limit(20).toArray();
```

## $slice with the Aggregation Pipeline

In aggregation, use the `$slice` expression operator (note: different syntax):

```javascript
db.posts.aggregate([
  { $match: { status: "published" } },
  {
    $project: {
      title: 1,
      recentComments: { $slice: ["$comments", -3] }
    }
  }
])
```

The aggregation `$slice` takes an array expression as the first argument, followed by the count (or skip and count).

## What $slice Does Not Do

- It does not filter elements by condition - use `$filter` (aggregation) or `$elemMatch` for that.
- It does not affect the original document - only the returned result.
- It works on arrays only; it cannot be applied to scalar fields.

## Common Mistakes

- Confusing the query projection `$slice` syntax with the aggregation `$slice` expression syntax.
- Using `$slice: 0` - this returns an empty array.
- Expecting `$slice` to work in the filter (first argument to `find()`) - it only works in projections.

## Summary

The `$slice` projection operator returns a subset of array elements: use positive numbers for the first N, negative numbers for the last N, and `[skip, limit]` for a specific range. It is ideal for previewing large arrays, showing recent items, and implementing array-level pagination. In aggregation pipelines, use `$slice` as an expression with slightly different syntax.
