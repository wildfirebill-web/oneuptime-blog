# How to Use $slice Modifier to Limit Array Size After Push in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Modifier, Update Operator, Document

Description: Learn how MongoDB's $slice modifier limits an array to a fixed number of elements after a $push operation, enabling bounded queues and rolling windows.

---

## What Is the $slice Modifier?

The `$slice` modifier is used with `$push` and `$each` to limit the size of an array after new elements are appended. It lets you maintain a rolling window of the most recent N items, implement capped arrays, or keep only the top N sorted results - all in a single atomic operation.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  {
    $push: {
      <arrayField>: {
        $each: [<value>],
        $slice: <number>
      }
    }
  }
)
```

`$slice` requires `$each` to be present. The slice value determines which elements to keep.

## $slice Values Explained

| Value | Behavior |
|-------|----------|
| Positive `n` | Keep the first `n` elements |
| Negative `-n` | Keep the last `n` elements |
| `0` | Clears the array entirely |

## Example: Keeping the Last 5 Logs

```javascript
db.servers.updateOne(
  { _id: "server-01" },
  {
    $push: {
      recentLogs: {
        $each: [{ ts: new Date(), msg: "CPU spike detected" }],
        $slice: -5
      }
    }
  }
)
```

After each push, the array retains only the 5 most recent log entries.

## Example: Keeping the First 10 Registrations

```javascript
db.events.updateOne(
  { _id: "hackathon-2024" },
  {
    $push: {
      earlyRegistrants: {
        $each: [{ name: "Alice", registeredAt: new Date() }],
        $slice: 10
      }
    }
  }
)
```

New registrants after the first 10 are discarded.

## Combining $slice with $sort

Push new elements, sort, and keep only the top N.

```javascript
db.leaderboards.updateOne(
  { _id: "weekly" },
  {
    $push: {
      topScores: {
        $each: [{ player: "Charlie", score: 1200 }],
        $sort: { score: -1 },
        $slice: 10
      }
    }
  }
)
```

After sorting descending, the array is trimmed to 10 entries.

## Truncating Without Pushing New Elements

You can truncate an existing array by passing an empty `$each`.

```javascript
db.feeds.updateOne(
  { _id: "user-feed-1" },
  {
    $push: {
      posts: {
        $each: [],
        $slice: -20
      }
    }
  }
)
```

This keeps the last 20 posts without adding any new ones.

## Using $slice to Clear an Array

```javascript
db.sessions.updateOne(
  { _id: "session-1" },
  {
    $push: {
      history: {
        $each: [],
        $slice: 0
      }
    }
  }
)
// history becomes []
```

## Slice vs Capped Collections

`$slice` gives you bounded arrays inside documents, while capped collections limit the overall collection size. Use `$slice` when only certain array fields need size limits rather than the entire document.

## Summary

The `$slice` modifier is the simplest way to keep MongoDB arrays bounded. Combined with `$each` and optionally `$sort`, it enables rolling queues, top-N leaderboards, and fixed-size activity feeds - all maintained atomically without separate cleanup operations.
