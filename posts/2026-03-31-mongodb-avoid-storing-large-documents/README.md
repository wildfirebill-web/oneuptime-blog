# How to Avoid Storing Large Documents (Near 16MB Limit) in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Document, Anti-Pattern, GridFS, Schema Design

Description: Learn why approaching MongoDB's 16MB document limit causes performance problems and how to refactor schemas using references, GridFS, or pagination.

---

MongoDB enforces a 16MB maximum document size. While hitting this hard limit is relatively rare, documents approaching even 1-2MB already cause significant performance degradation. Large documents increase memory pressure, slow query responses, and create write amplification on updates.

## Why Large Documents Cause Problems

Even before hitting 16MB, large documents create real issues:

- **Network overhead** - every query that touches the document transfers all that data
- **Working set pressure** - large documents consume more RAM in the WiredTiger cache
- **Update amplification** - updating one field in a 5MB document rewrites the entire document
- **Aggregation limits** - the 100MB aggregation limit is hit faster with large documents

## Common Causes of Document Bloat

The most common cause is unbounded embedded arrays - storing all related records as array elements in a parent document:

```javascript
// BAD - all comments embedded in post document
// After 10,000 comments, this document approaches or exceeds 16MB
const post = {
  _id: postId,
  title: "My Blog Post",
  content: "...",
  comments: [
    { userId: "u1", text: "Great post!", createdAt: new Date() },
    { userId: "u2", text: "Thanks for sharing", createdAt: new Date() },
    // ... 9,998 more comments
  ]
};
```

## Solution 1 - Reference Pattern

Store related data in a separate collection with a reference to the parent:

```javascript
// GOOD - comments in separate collection
const comment = {
  _id: new ObjectId(),
  postId: postId,          // reference to parent
  userId: "u1",
  text: "Great post!",
  createdAt: new Date()
};

await db.collection('comments').insertOne(comment);
await db.collection('comments').createIndex({ postId: 1, createdAt: -1 });

// Fetch comments separately when needed
const comments = await db.collection('comments')
  .find({ postId: postId })
  .sort({ createdAt: -1 })
  .limit(20)
  .toArray();
```

## Solution 2 - Subset Pattern

Keep a small, fixed subset of the most relevant data embedded and reference the rest:

```javascript
// Store only the last 5 comments embedded, reference the rest
const post = {
  _id: postId,
  title: "My Blog Post",
  recentComments: [
    // last 5 comments only
  ],
  commentCount: 10234
};
```

## Solution 3 - GridFS for Binary Data

For large files (images, videos, PDFs), use GridFS rather than storing binary data in documents:

```javascript
const { GridFSBucket } = require('mongodb');
const fs = require('fs');

async function storeFile(db, filePath, filename) {
  const bucket = new GridFSBucket(db);
  const uploadStream = bucket.openUploadStream(filename, {
    metadata: { uploadedAt: new Date() }
  });
  fs.createReadStream(filePath).pipe(uploadStream);
  return new Promise((resolve, reject) => {
    uploadStream.on('finish', () => resolve(uploadStream.id));
    uploadStream.on('error', reject);
  });
}

async function retrieveFile(db, fileId, destPath) {
  const bucket = new GridFSBucket(db);
  const downloadStream = bucket.openDownloadStream(fileId);
  downloadStream.pipe(fs.createWriteStream(destPath));
}
```

GridFS splits files into 255KB chunks stored in a `fs.chunks` collection, keeping each chunk well within limits.

## Detecting Large Documents

Find documents that are approaching problematic sizes:

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      docSize: { $bsonSize: "$$ROOT" }
    }
  },
  { $match: { docSize: { $gt: 500000 } } },   // over 500KB
  { $sort: { docSize: -1 } },
  { $limit: 20 }
]);
```

## Summary

Documents that approach MongoDB's 16MB limit cause performance problems well before the hard limit is reached. Use the reference pattern to move unbounded arrays to separate collections, the subset pattern to embed only the most frequently accessed data, and GridFS for binary content. Monitor document sizes with `$bsonSize` to catch growing documents before they become a production problem.
