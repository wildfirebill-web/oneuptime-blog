# How to Design a Social Network Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Social Network, Data Modeling, Graph

Description: Learn how to design an efficient MongoDB schema for a social network including users, posts, follows, likes, and comments using embedding and referencing strategies.

---

## Core Collections

A social network typically involves users, posts, follows (connections), comments, and likes. MongoDB's flexible schema allows embedding or referencing based on access patterns.

## Users Collection

```javascript
db.users.insertOne({
  _id: ObjectId("usr001"),
  username: "alice",
  displayName: "Alice Smith",
  email: "alice@example.com",
  passwordHash: "$2b$10$...",
  bio: "Software engineer and coffee enthusiast",
  avatarUrl: "https://cdn.example.com/avatars/usr001.jpg",
  followerCount: 1240,
  followingCount: 310,
  postCount: 87,
  createdAt: new Date("2024-01-15"),
  settings: {
    notifications: { likes: true, comments: true, follows: true },
    privacy: "public"
  }
})
```

Counter fields (`followerCount`, `followingCount`) are denormalized for fast display without aggregation.

## Posts Collection

```javascript
db.posts.insertOne({
  _id: ObjectId("post001"),
  authorId: ObjectId("usr001"),
  authorUsername: "alice",
  authorAvatarUrl: "https://cdn.example.com/avatars/usr001.jpg",
  content: "Just shipped a new feature! #mongodb #engineering",
  mediaUrls: ["https://cdn.example.com/media/post001-1.jpg"],
  hashtags: ["mongodb", "engineering"],
  likeCount: 42,
  commentCount: 8,
  visibility: "public",
  createdAt: new Date("2026-03-30T14:22:00Z")
})

db.posts.createIndex({ authorId: 1, createdAt: -1 })
db.posts.createIndex({ hashtags: 1, createdAt: -1 })
```

## Follows Collection

```javascript
db.follows.insertOne({
  _id: ObjectId(),
  followerId: ObjectId("usr002"),
  followeeId: ObjectId("usr001"),
  createdAt: new Date()
})

db.follows.createIndex({ followerId: 1, followeeId: 1 }, { unique: true })
db.follows.createIndex({ followeeId: 1, followerId: 1 })
```

## Comments - Embedded vs Referenced

```javascript
// Option A: embedded (best for low-volume comment threads, max ~100 comments)
db.posts.updateOne(
  { _id: ObjectId("post001") },
  {
    $push: {
      comments: {
        commentId: ObjectId(),
        authorId: ObjectId("usr002"),
        authorUsername: "bob",
        content: "Congrats on the release!",
        likeCount: 3,
        createdAt: new Date()
      }
    },
    $inc: { commentCount: 1 }
  }
)

// Option B: separate collection (best for high-volume threads)
db.comments.insertOne({
  _id: ObjectId(),
  postId: ObjectId("post001"),
  parentCommentId: null,
  authorId: ObjectId("usr002"),
  authorUsername: "bob",
  content: "Congrats!",
  likeCount: 3,
  createdAt: new Date()
})

db.comments.createIndex({ postId: 1, createdAt: -1 })
db.comments.createIndex({ parentCommentId: 1 })
```

## Likes Collection

```javascript
db.likes.insertOne({
  _id: ObjectId(),
  userId: ObjectId("usr002"),
  targetType: "post",
  targetId: ObjectId("post001"),
  createdAt: new Date()
})

db.likes.createIndex({ userId: 1, targetId: 1, targetType: 1 }, { unique: true })
db.likes.createIndex({ targetId: 1, targetType: 1 })
```

## Newsfeed Query

```javascript
async function getHomeFeed(db, userId, limit = 20, beforeDate) {
  const follows = await db.collection("follows")
    .find({ followerId: userId })
    .project({ followeeId: 1 })
    .toArray()

  const feedAuthorIds = follows.map(f => f.followeeId)
  feedAuthorIds.push(userId)

  const query = { authorId: { $in: feedAuthorIds }, visibility: "public" }
  if (beforeDate) query.createdAt = { $lt: beforeDate }

  return db.collection("posts")
    .find(query)
    .sort({ createdAt: -1 })
    .limit(limit)
    .toArray()
}
```

## Notifications Schema

```javascript
db.notifications.insertOne({
  recipientId: ObjectId("usr001"),
  actorId: ObjectId("usr002"),
  actorUsername: "bob",
  type: "like",
  targetType: "post",
  targetId: ObjectId("post001"),
  isRead: false,
  createdAt: new Date()
})

db.notifications.createIndex({ recipientId: 1, isRead: 1, createdAt: -1 })
```

## Summary

Design a MongoDB social network schema using separate collections for users, posts, follows, comments, likes, and notifications. Denormalize frequently displayed fields such as username and avatar inside posts for feed performance. Use counter fields for follower and post counts, model the social graph in a dedicated follows collection, and build indexes aligned with your feed, profile, and notification query patterns.
