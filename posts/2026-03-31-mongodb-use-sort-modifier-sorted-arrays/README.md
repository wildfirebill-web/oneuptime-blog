# How to Use $sort Modifier with $push to Maintain Sorted Arrays in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Modifier, Update Operator, Document

Description: Learn how the $sort modifier with $push keeps MongoDB arrays automatically sorted after every insertion, perfect for leaderboards, priority queues, and ranked lists.

---

## What Is the $sort Modifier?

The `$sort` modifier is used with `$push` and `$each` to sort the entire array after new elements are appended. Rather than sorting the array in application code after retrieval, `$sort` maintains the order at the database level so every read always returns a sorted array.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  {
    $push: {
      <arrayField>: {
        $each: [<value>],
        $sort: <sortSpecification>
      }
    }
  }
)
```

`$sort` requires `$each`. Use `1` for ascending, `-1` for descending.

## Example: Maintaining a Sorted Leaderboard

```javascript
db.leaderboards.updateOne(
  { game: "chess" },
  {
    $push: {
      rankings: {
        $each: [{ player: "Alice", score: 1450 }],
        $sort: { score: -1 }
      }
    }
  }
)
```

After the push, the `rankings` array is sorted from highest to lowest score.

## Sorting an Array of Scalars

For arrays of numbers or strings, use `1` or `-1` directly.

```javascript
db.config.updateOne(
  { _id: "priority-list" },
  {
    $push: {
      priorities: {
        $each: [5, 2, 8],
        $sort: 1
      }
    }
  }
)
// priorities is kept in ascending order
```

## Sorting by Multiple Fields

```javascript
db.tasks.updateOne(
  { _id: "queue-1" },
  {
    $push: {
      jobs: {
        $each: [{ priority: 1, createdAt: new Date(), name: "job-X" }],
        $sort: { priority: -1, createdAt: 1 }
      }
    }
  }
)
```

Sorts by priority descending, then by creation time ascending as a tiebreaker.

## Combining $sort with $slice for a Top-N List

```javascript
db.scores.updateOne(
  { _id: "weekly-top-10" },
  {
    $push: {
      entries: {
        $each: [{ name: "Bob", score: 980 }],
        $sort: { score: -1 },
        $slice: 10
      }
    }
  }
)
```

New entries are merged in, the array is sorted, and only the top 10 are kept.

## Sorting the Entire Array Without Inserting New Elements

Pass an empty `$each` to re-sort an existing array without adding elements.

```javascript
db.products.updateOne(
  { _id: "cat-electronics" },
  {
    $push: {
      featuredItems: {
        $each: [],
        $sort: { price: 1 }
      }
    }
  }
)
```

## Nested Field Sorting

Sort embedded objects by a deeply nested field using dot notation.

```javascript
db.playlists.updateOne(
  { _id: "playlist-1" },
  {
    $push: {
      tracks: {
        $each: [{ meta: { duration: 240 }, title: "New Song" }],
        $sort: { "meta.duration": 1 }
      }
    }
  }
)
```

## When to Use $sort vs Application-Side Sorting

| Approach | Pros | Cons |
|----------|------|------|
| $sort modifier | Always sorted, atomic | Sorts entire array on each write |
| Application sorting | Flexible on read | Extra computation, stale on concurrent writes |

Use `$sort` when reads outnumber writes and you always need a consistent order.

## Summary

The `$sort` modifier with `$push` ensures MongoDB arrays remain in the desired order after every insertion. Paired with `$slice`, it efficiently maintains bounded, sorted structures like leaderboards, priority queues, and top-N lists entirely within the database layer.
