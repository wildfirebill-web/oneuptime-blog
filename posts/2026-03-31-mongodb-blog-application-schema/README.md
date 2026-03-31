# How to Model a Blog Application Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Blog, Document Model, Aggregation

Description: Learn how to design a blog application schema in MongoDB using embedded documents and references to handle posts, authors, comments, and tags efficiently.

---

## Overview of the Blog Data Model

A blog application has several interconnected entities: authors, posts, comments, tags, and categories. MongoDB's flexible document model lets you embed related data directly or reference it from separate collections depending on access patterns.

The key design decisions are:
- Embed frequently co-read data (tags, summary author info)
- Reference independently growing data (comments, full author profiles)
- Avoid unbounded arrays in a single document

## The Authors Collection

Authors have stable profile information. Store each author as a standalone document:

```javascript
db.authors.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
  username: "jane_doe",
  displayName: "Jane Doe",
  email: "jane@example.com",
  bio: "Software engineer and technical writer.",
  avatarUrl: "https://cdn.example.com/avatars/jane.jpg",
  createdAt: new Date("2024-01-15")
});
```

## The Posts Collection

Posts embed a lightweight author summary and an array of tags. Comments are stored in a separate collection to avoid unbounded growth:

```javascript
db.posts.insertOne({
  _id: ObjectId("64b2c3d4e5f6a7b8c9d0e2f3"),
  slug: "getting-started-with-mongodb",
  title: "Getting Started with MongoDB",
  body: "MongoDB is a document-oriented database...",
  excerpt: "A beginner's guide to MongoDB data modeling.",
  status: "published",
  author: {
    _id: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
    username: "jane_doe",
    displayName: "Jane Doe",
    avatarUrl: "https://cdn.example.com/avatars/jane.jpg"
  },
  tags: ["mongodb", "nosql", "database", "tutorial"],
  category: "Database",
  viewCount: 1542,
  likeCount: 87,
  commentCount: 23,
  publishedAt: new Date("2024-03-10"),
  updatedAt: new Date("2024-03-12"),
  seo: {
    metaTitle: "Getting Started with MongoDB - Tutorial",
    metaDescription: "Learn MongoDB basics with practical examples."
  }
});
```

## The Comments Collection

Store comments separately to keep post documents small:

```javascript
db.comments.insertOne({
  _id: ObjectId("64c3d4e5f6a7b8c9d0e3f4a5"),
  postId: ObjectId("64b2c3d4e5f6a7b8c9d0e2f3"),
  author: {
    _id: ObjectId("64a9b8c7d6e5f4a3b2c1d0e9"),
    username: "reader_bob",
    displayName: "Bob Smith",
    avatarUrl: "https://cdn.example.com/avatars/bob.jpg"
  },
  body: "Great article! Very helpful for beginners.",
  parentId: null,
  likeCount: 5,
  createdAt: new Date("2024-03-11T14:30:00Z"),
  status: "approved"
});
```

Set `parentId` to the `_id` of another comment to support nested replies.

## Indexes

Create indexes to support common query patterns:

```javascript
// Fetch published posts sorted by date
db.posts.createIndex({ status: 1, publishedAt: -1 });

// Look up posts by slug
db.posts.createIndex({ slug: 1 }, { unique: true });

// Filter posts by tag
db.posts.createIndex({ tags: 1 });

// Comments for a post, newest first
db.comments.createIndex({ postId: 1, createdAt: -1 });

// Filter by category and sort
db.posts.createIndex({ category: 1, publishedAt: -1 });
```

## Querying the Blog Feed

Fetch the 10 most recent published posts with projection to avoid loading the full body:

```javascript
db.posts.find(
  { status: "published" },
  { body: 0 }
).sort({ publishedAt: -1 }).limit(10);
```

## Aggregating Popular Posts by Tag

```javascript
db.posts.aggregate([
  { $match: { status: "published" } },
  { $unwind: "$tags" },
  {
    $group: {
      _id: "$tags",
      postCount: { $sum: 1 },
      totalViews: { $sum: "$viewCount" }
    }
  },
  { $sort: { totalViews: -1 } },
  { $limit: 10 }
]);
```

## Updating Counts Safely

Use atomic increments to update view and comment counts:

```javascript
// Increment view count
db.posts.updateOne(
  { _id: ObjectId("64b2c3d4e5f6a7b8c9d0e2f3") },
  { $inc: { viewCount: 1 } }
);

// Increment comment count when a new comment is added
db.posts.updateOne(
  { _id: postId },
  { $inc: { commentCount: 1 } }
);
```

## Denormalization Trade-offs

The author subdocument in posts is a denormalized snapshot. If an author updates their display name, you must decide whether to backfill existing posts. A common approach is to treat the embedded author as an immutable publication record and only update future posts.

For display-critical fields like `avatarUrl`, run a background job to refresh the embedded copy periodically:

```javascript
db.posts.updateMany(
  { "author._id": authorId },
  { $set: { "author.avatarUrl": newAvatarUrl } }
);
```

## Summary

A well-designed MongoDB blog schema embeds stable, frequently co-read data like author summaries and tags directly in post documents, while storing growing collections like comments in separate documents linked by reference. Counting fields such as `viewCount` and `commentCount` should be maintained with atomic increments. This schema supports fast feed queries, tag filtering, and comment threading without expensive joins.
