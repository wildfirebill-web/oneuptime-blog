# How to Implement the Outlier Pattern in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Outlier Pattern, Schema Design, Performance

Description: Learn how to implement the outlier pattern in MongoDB to handle rare documents with unusually large data without degrading performance for typical documents.

---

## What Is the Outlier Pattern?

The outlier pattern handles the case where most documents follow a standard structure, but a small percentage of documents (outliers) have significantly more data than typical. Instead of designing the entire schema around rare outliers, the pattern treats them specially while keeping the standard path fast.

A classic example: most blog posts have a few comments, but viral posts may have millions. Designing the schema for the typical case keeps performance high for 99% of queries.

## The Problem: Designing for Outliers

If you embed comments in a blog post document, most posts work fine:

```javascript
// Typical post - works great
{
  _id: ObjectId("post001"),
  title: "MongoDB Tips",
  comments: [
    { author: "alice", body: "Great post!" },
    { author: "bob", body: "Thanks!" }
  ]
}
```

But a viral post could accumulate millions of comments, hitting the 16MB document limit and causing write amplification on every comment addition.

## The Outlier Pattern: Flag and Overflow

Mark outlier documents with a flag and store overflow data in a separate collection:

```javascript
// Normal post document
{
  _id: ObjectId("post001"),
  title: "MongoDB Tips",
  hasOverflow: false,
  comments: [
    { author: "alice", body: "Great post!" },
    { author: "bob", body: "Thanks!" }
  ]
}

// Viral post document (outlier)
{
  _id: ObjectId("viral001"),
  title: "10 Things You Didn't Know About MongoDB",
  hasOverflow: true,
  commentCount: 125000,
  comments: [
    // Only first N comments embedded (e.g., most recent 100)
    { author: "user1", body: "Amazing!", createdAt: ISODate("2026-03-31T00:00:00Z") }
    // ... up to 100 recent comments
  ]
}

// Overflow collection for outlier comments
{
  _id: ObjectId("..."),
  postId: ObjectId("viral001"),
  comments: [
    { author: "user101", body: "Wow!", createdAt: ISODate("2026-03-01T00:00:00Z") }
    // ... batches of older comments
  ]
}
```

## Application Logic for Outlier Handling

```javascript
async function getComments(postId, page = 0, limit = 20) {
  const post = await db.posts.findOne({ _id: postId });

  if (!post.hasOverflow) {
    // Standard path: all comments are embedded
    return post.comments.slice(page * limit, (page + 1) * limit);
  }

  // Outlier path: paginate from overflow collection
  return db.postOverflowComments.aggregate([
    { $match: { postId: postId } },
    { $unwind: "$comments" },
    { $sort: { "comments.createdAt": -1 } },
    { $skip: page * limit },
    { $limit: limit },
    { $replaceRoot: { newRoot: "$comments" } }
  ]).toArray();
}
```

## Detecting and Marking Outliers

Detect outliers based on a threshold and update the flag:

```javascript
// When adding a comment, check if we've exceeded the threshold
async function addComment(postId, comment) {
  const post = await db.posts.findOneAndUpdate(
    { _id: postId },
    {
      $push: {
        comments: {
          $each: [comment],
          $slice: -100 // Keep only last 100 comments embedded
        }
      },
      $inc: { commentCount: 1 }
    },
    { returnDocument: "after" }
  );

  if (post.commentCount > 100 && !post.hasOverflow) {
    // Mark as outlier
    await db.posts.updateOne({ _id: postId }, { $set: { hasOverflow: true } });
  }

  // Always insert to overflow collection for outliers
  if (post.hasOverflow) {
    await db.postOverflowComments.updateOne(
      { postId: postId, overflowCount: { $lt: 1000 } },
      { $push: { comments: comment }, $inc: { overflowCount: 1 } },
      { upsert: true }
    );
  }
}
```

## Real-World Applications

```text
Use Case                        | Typical        | Outlier
--------------------------------|----------------|------------------------
Blog comments                   | < 50 comments  | Viral post (millions)
E-commerce product reviews      | < 200 reviews  | Best-seller (thousands)
Social network followers         | < 500          | Celebrities (millions)
Order history                   | < 100 orders   | Enterprise accounts
```

## Indexing Considerations

```javascript
// Index to quickly find overflow documents for a post
db.postOverflowComments.createIndex({ postId: 1 })

// Index to filter non-outliers in the standard path
db.posts.createIndex({ hasOverflow: 1, createdAt: -1 })
```

## Summary

The outlier pattern separates the schema design for the common case from the rare outlier case. Use a boolean flag (`hasOverflow`, `hasExtended`) to mark documents that exceed a threshold, embed a limited set of data for all documents, and store excess data in an overflow collection. The application branches on the flag to use the efficient embedded path for typical documents and the overflow path for outliers, keeping the standard code path fast for the vast majority of operations.
