# How to Model a Social Media Feed in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Social Media, Feed, Fanout, Document Model

Description: Learn how to design a scalable social media feed in MongoDB using push and pull fanout strategies with practical schema examples and index recommendations.

---

## The Challenge of Social Feeds

Social feeds are among the most demanding read workloads in any application. When a user opens their feed, the system must quickly retrieve posts from everyone they follow, ranked by recency or relevance. MongoDB's document model handles this well with the right schema and indexing strategy.

There are two main approaches to feed delivery: pull-on-read (fan-out on read) and push-on-write (fan-out on write). Most production systems use a hybrid.

## The Users Collection

```javascript
db.users.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
  username: "alice",
  displayName: "Alice Chen",
  avatarUrl: "https://cdn.example.com/avatars/alice.jpg",
  followerCount: 1200,
  followingCount: 340,
  createdAt: new Date("2023-06-01")
});
```

## The Posts Collection

Posts are written once by their author. Each post stores a lightweight author snapshot:

```javascript
db.posts.insertOne({
  _id: ObjectId("64b1c2d3e4f5a6b7c8d9e0f1"),
  authorId: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
  author: {
    username: "alice",
    displayName: "Alice Chen",
    avatarUrl: "https://cdn.example.com/avatars/alice.jpg"
  },
  content: "Just shipped a new feature using MongoDB aggregation pipelines!",
  mediaUrls: ["https://cdn.example.com/posts/img1.jpg"],
  likeCount: 142,
  commentCount: 18,
  shareCount: 7,
  createdAt: new Date("2024-05-20T09:00:00Z"),
  visibility: "public"
});
```

## The Follows Collection

Track follow relationships in a dedicated collection:

```javascript
db.follows.insertOne({
  followerId: ObjectId("64c2d3e4f5a6b7c8d9e0f1a2"),
  followeeId: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
  createdAt: new Date("2024-01-10")
});

// Index for fast lookup of who a user follows
db.follows.createIndex({ followerId: 1, createdAt: -1 });
// Index for fast lookup of a user's followers
db.follows.createIndex({ followeeId: 1 });
```

## Fan-out on Read (Pull Model)

In the pull model, the feed is computed at read time. When Alice loads her feed, the app queries posts from all users she follows:

```javascript
async function getFeed(db, userId, page = 0, limit = 20) {
  // Step 1: get list of followed users
  const follows = await db.follows
    .find({ followerId: userId })
    .project({ followeeId: 1 })
    .toArray();

  const followeeIds = follows.map(f => f.followeeId);
  followeeIds.push(userId); // include own posts

  // Step 2: query recent posts from those users
  return db.posts
    .find({
      authorId: { $in: followeeIds },
      visibility: "public"
    })
    .sort({ createdAt: -1 })
    .skip(page * limit)
    .limit(limit)
    .toArray();
}
```

This scales well for users with few follows but degrades when following thousands of accounts.

## Fan-out on Write (Push Model)

In the push model, each new post is written into the feed collections of all followers:

```javascript
// Feed inbox collection
db.feedItems.insertOne({
  userId: ObjectId("64c2d3e4f5a6b7c8d9e0f1a2"), // the follower
  postId: ObjectId("64b1c2d3e4f5a6b7c8d9e0f1"),
  authorId: ObjectId("64a1b2c3d4e5f6a7b8c9d0e1"),
  createdAt: new Date("2024-05-20T09:00:00Z")
});

// Index for reading a user's feed
db.feedItems.createIndex({ userId: 1, createdAt: -1 });
```

When a post is published, fan it out to all followers:

```javascript
async function fanOutPost(db, postId, authorId, createdAt) {
  const followers = await db.follows
    .find({ followeeId: authorId })
    .project({ followerId: 1 })
    .toArray();

  const feedItems = followers.map(f => ({
    userId: f.followerId,
    postId,
    authorId,
    createdAt
  }));

  if (feedItems.length > 0) {
    await db.feedItems.insertMany(feedItems, { ordered: false });
  }
}
```

Reading the feed becomes a simple indexed lookup - extremely fast.

## Reading the Feed (Push Model)

```javascript
async function getFeedPush(db, userId, before = new Date(), limit = 20) {
  const feedItems = await db.feedItems
    .find({ userId, createdAt: { $lt: before } })
    .sort({ createdAt: -1 })
    .limit(limit)
    .toArray();

  const postIds = feedItems.map(f => f.postId);
  return db.posts.find({ _id: { $in: postIds } }).toArray();
}
```

## Handling Likes and Reactions

Store reactions in a dedicated collection to avoid unbounded arrays:

```javascript
db.reactions.insertOne({
  postId: ObjectId("64b1c2d3e4f5a6b7c8d9e0f1"),
  userId: ObjectId("64c2d3e4f5a6b7c8d9e0f1a2"),
  type: "like",
  createdAt: new Date()
});

db.reactions.createIndex({ postId: 1, userId: 1 }, { unique: true });

// Atomic increment on the post
db.posts.updateOne(
  { _id: postId },
  { $inc: { likeCount: 1 } }
);
```

## Choosing Between Pull and Push

| Factor | Pull (Fan-out on Read) | Push (Fan-out on Write) |
|---|---|---|
| Write cost | Low | High (N followers) |
| Read cost | High ($in query) | Low (indexed lookup) |
| Best for | Low follower counts | High read:write ratio |
| Celebrity problem | Fine | Expensive (millions of writes) |

A hybrid strategy is common: use push for regular users and fall back to pull for celebrity accounts with millions of followers.

## TTL Index for Feed Cleanup

Keep the feed inbox fresh by expiring old entries automatically:

```javascript
db.feedItems.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 60 * 60 * 24 * 30 } // 30 days
);
```

## Summary

Modeling a social media feed in MongoDB requires choosing between fan-out on read, fan-out on write, or a hybrid approach based on your follower distribution and read:write ratio. Push-model schemas using a feedItems collection with compound indexes deliver the fastest read performance. Reactions and likes belong in separate collections with atomic counter updates on the post document to keep writes safe and documents bounded.
