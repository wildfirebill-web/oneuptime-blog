# How to Handle Unbounded Array Growth in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Performance, Array, Data Modeling

Description: Learn how to detect and fix unbounded array growth in MongoDB documents using the Bucket Pattern, references, and capped arrays to keep documents performant.

---

Unbounded arrays in MongoDB documents are a common performance trap. When an array field grows without limits, documents eventually exceed the 16 MB BSON size limit, indexes on array fields become bloated, and queries slow down significantly. This guide shows practical techniques to prevent and fix unbounded array growth.

## Why Unbounded Arrays Are Dangerous

Consider a social media post document where every like and comment is stored as an array element:

```javascript
{
  _id: ObjectId("..."),
  content: "Hello world",
  likes: [userId1, userId2, userId3, ...], // could grow to millions
  comments: [{ text: "...", author: "..." }, ...] // no upper bound
}
```

When this document hits the 16 MB BSON limit, writes will fail. Before that, query and index performance degrades as MongoDB must load entire large documents.

## Strategy 1 - Use the Bucket Pattern

Instead of one document with an unbounded array, split data into fixed-size buckets. This is ideal for time-series or append-only data:

```javascript
// Instead of one post with all likes, use like-buckets
{
  _id: ObjectId("..."),
  postId: ObjectId("abc123"),
  bucket: 1,
  count: 200,
  likes: [userId1, userId2, ..., userId200] // capped at 200
}
```

Create new bucket documents when the current one is full:

```javascript
db.likeBuckets.updateOne(
  { postId: postId, count: { $lt: 200 } },
  {
    $push: { likes: newUserId },
    $inc: { count: 1 }
  },
  { upsert: true }
)
```

## Strategy 2 - Reference Instead of Embed

Move high-cardinality data to a separate collection and reference it:

```javascript
// Post document stays small
{
  _id: ObjectId("abc123"),
  content: "Hello world",
  likeCount: 15032
}

// Separate likes collection
{
  _id: ObjectId("..."),
  postId: ObjectId("abc123"),
  userId: ObjectId("user456"),
  likedAt: ISODate("2026-03-31T10:00:00Z")
}
```

```javascript
db.likes.createIndex({ postId: 1, userId: 1 }, { unique: true })
```

## Strategy 3 - Cap Arrays with $slice

For cases where you only need the most recent N elements, use `$slice` during push operations:

```javascript
// Keep only the latest 100 activity entries
db.users.updateOne(
  { _id: userId },
  {
    $push: {
      recentActivity: {
        $each: [{ action: "login", ts: new Date() }],
        $slice: -100
      }
    }
  }
)
```

This is useful for activity feeds, audit logs within a document, or notification previews.

## Strategy 4 - Use TTL Indexes on Subdocument Collections

If array data is time-bound, store it in a separate collection with a TTL index:

```javascript
db.events.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })

db.events.insertOne({
  userId: ObjectId("..."),
  type: "page_view",
  url: "/dashboard",
  expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // 30 days
})
```

## Detecting Problematic Documents

Use the aggregation pipeline to find documents with oversized arrays:

```javascript
db.posts.aggregate([
  {
    $project: {
      likeCount: { $size: "$likes" },
      commentCount: { $size: "$comments" }
    }
  },
  { $match: { likeCount: { $gt: 1000 } } },
  { $sort: { likeCount: -1 } },
  { $limit: 10 }
])
```

## Choosing the Right Strategy

- Use the **Bucket Pattern** for time-series data with predictable chunk sizes.
- Use **references** for many-to-many or high-cardinality relationships.
- Use **$slice** when only the latest N records matter.
- Use **TTL indexes** for temporary or expiring data.

The key principle is that any array that could grow beyond a few hundred elements should be moved to a separate collection or bucketed.

## Summary

Unbounded arrays are one of the most common MongoDB performance issues. The Bucket Pattern splits data into fixed-size documents, referencing moves arrays to dedicated collections, and `$slice` caps array length at write time. Regularly audit document sizes in production to catch unbounded growth early before it affects application performance.
