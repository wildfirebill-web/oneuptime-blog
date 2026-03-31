# How to Use $each Modifier with $push in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Modifier, Update Operator, Document

Description: Learn how the $each modifier with $push lets you append multiple elements to a MongoDB array in a single operation, and how it unlocks other powerful modifiers.

---

## What Is the $each Modifier?

The `$each` modifier is used alongside `$push` (and `$addToSet`) to append multiple values to an array in a single update operation. Without `$each`, `$push` would append the entire array as a single nested element. `$each` also enables other array modifiers like `$slice`, `$sort`, and `$position` that require it as a prerequisite.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  {
    $push: {
      <arrayField>: {
        $each: [<value1>, <value2>, ...]
      }
    }
  }
)
```

## Example: Appending Multiple Scores

```javascript
// Without $each - appends the array as a single element
db.students.updateOne(
  { _id: 1 },
  { $push: { scores: [85, 90, 95] } }
)
// scores becomes: [..., [85, 90, 95]]  -- nested array!

// With $each - appends each element individually
db.students.updateOne(
  { _id: 1 },
  { $push: { scores: { $each: [85, 90, 95] } } }
)
// scores becomes: [..., 85, 90, 95]  -- correct
```

## Appending Objects to an Array

```javascript
db.orders.updateOne(
  { _id: "order-1" },
  {
    $push: {
      items: {
        $each: [
          { sku: "A1", qty: 2, price: 10.00 },
          { sku: "B2", qty: 1, price: 25.00 }
        ]
      }
    }
  }
)
```

## Combining $each with $sort

Sort the array after inserting new elements.

```javascript
db.leaderboards.updateOne(
  { _id: "game-1" },
  {
    $push: {
      scores: {
        $each: [
          { player: "Alice", score: 980 },
          { player: "Bob", score: 870 }
        ],
        $sort: { score: -1 }
      }
    }
  }
)
```

## Combining $each with $slice

Append elements and keep only the most recent N.

```javascript
db.feeds.updateOne(
  { userId: "user-1" },
  {
    $push: {
      recentActivity: {
        $each: [{ action: "login", ts: new Date() }],
        $slice: -50
      }
    }
  }
)
```

The negative slice `-50` retains the last 50 elements, effectively capping the array size.

## Combining $each with $position

Insert elements at a specific index.

```javascript
db.playlists.updateOne(
  { _id: "playlist-1" },
  {
    $push: {
      songs: {
        $each: ["song-new-1", "song-new-2"],
        $position: 0
      }
    }
  }
)
// Inserts both songs at the beginning of the array
```

## Using $each with $addToSet

`$each` also works with `$addToSet` to add multiple unique values.

```javascript
db.articles.updateOne(
  { _id: "article-1" },
  {
    $addToSet: {
      tags: { $each: ["mongodb", "database", "nosql"] }
    }
  }
)
```

Only values not already in `tags` are added.

## Summary

The `$each` modifier is essential when you need to push multiple elements to a MongoDB array without creating a nested array. It also unlocks the `$slice`, `$sort`, and `$position` modifiers, enabling powerful array management patterns like bounded queues, sorted leaderboards, and priority-based insertion all in a single atomic update.
