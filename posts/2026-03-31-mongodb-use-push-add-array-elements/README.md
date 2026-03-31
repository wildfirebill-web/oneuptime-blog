# How to Use $push to Add Elements to an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array Operator, Update Operator, Document

Description: Learn how MongoDB's $push operator appends elements to an array field, including appending single values, multiple values with $each, and combining with modifiers.

---

## What Is the $push Operator?

The `$push` operator appends a value to an array field in a MongoDB document. If the field does not exist, `$push` creates a new array with the appended value. It is one of the most commonly used array update operators and supports modifiers like `$each`, `$slice`, `$sort`, and `$position` for advanced control.

## Basic Syntax

```javascript
db.collection.updateOne(
  { <filter> },
  { $push: { <arrayField>: <value> } }
)
```

## Example: Appending a Single Value

```javascript
db.students.updateOne(
  { _id: "student-1" },
  { $push: { grades: 95 } }
)
```

If `grades` is `[85, 90]`, it becomes `[85, 90, 95]`.

## Example: Pushing an Object into an Array

```javascript
db.orders.updateOne(
  { _id: "order-42" },
  {
    $push: {
      items: { productId: "prod-7", quantity: 2, price: 19.99 }
    }
  }
)
```

## Creating the Array if It Does Not Exist

If `tags` does not exist on the document, `$push` creates it.

```javascript
db.articles.updateOne(
  { _id: "article-5" },
  { $push: { tags: "mongodb" } }
)
// tags field is created: ["mongodb"]
```

## Pushing Multiple Values with $each

To append multiple elements in a single operation, use `$each`.

```javascript
db.playlists.updateOne(
  { _id: "playlist-1" },
  {
    $push: {
      songs: {
        $each: ["song-a", "song-b", "song-c"]
      }
    }
  }
)
```

## Combining $push with $sort and $slice

Keep an array sorted and capped at a fixed length.

```javascript
db.leaderboards.updateOne(
  { game: "chess" },
  {
    $push: {
      topScores: {
        $each: [{ name: "Alice", score: 980 }],
        $sort: { score: -1 },
        $slice: 10
      }
    }
  }
)
```

This inserts a new score, sorts descending, and keeps only the top 10.

## Pushing at a Specific Position with $position

```javascript
db.queues.updateOne(
  { _id: "priority-queue" },
  {
    $push: {
      tasks: {
        $each: [{ id: "task-99", priority: "high" }],
        $position: 0
      }
    }
  }
)
```

This prepends the task to the front of the array.

## Bulk Appending to Multiple Documents

```javascript
db.groups.updateMany(
  { status: "active" },
  { $push: { log: { event: "daily-sync", ts: new Date() } } }
)
```

## Avoiding Duplicates

If you need unique elements, use `$addToSet` instead of `$push`. `$push` always appends regardless of existing values.

## Summary

The `$push` operator is the standard way to append values to MongoDB array fields. It handles single and multiple insertions, supports position control, and can maintain sorted, bounded arrays via modifiers. For unique-only insertions, combine `$push` with manual filtering or switch to `$addToSet`.
