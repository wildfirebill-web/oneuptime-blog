# How to Paginate Results with Mongoose in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Pagination, Query, Performance

Description: Learn how to implement offset-based and cursor-based pagination with Mongoose to efficiently serve large datasets through your MongoDB-backed API.

---

## Overview

Pagination is essential when working with large datasets. Mongoose supports two main pagination strategies: offset-based (skip/limit) for simple use cases, and cursor-based (keyset) for high-performance, consistent pagination on large collections.

## Offset-Based Pagination (Skip/Limit)

The simplest approach uses `skip` and `limit`:

```javascript
async function getPage(model, filter = {}, page = 1, limit = 20) {
  const skip = (page - 1) * limit;
  const [docs, total] = await Promise.all([
    model.find(filter).sort({ createdAt: -1 }).skip(skip).limit(limit).lean(),
    model.countDocuments(filter)
  ]);

  return {
    docs,
    total,
    page,
    pages:    Math.ceil(total / limit),
    hasNext:  page * limit < total,
    hasPrev:  page > 1
  };
}

// Usage
const result = await getPage(User, { active: true }, 2, 10);
```

## Adding Pagination as a Static Method

```javascript
const { Schema, model } = require('mongoose');

const postSchema = new Schema({ title: String, createdAt: { type: Date, default: Date.now } });

postSchema.statics.paginate = async function(filter = {}, options = {}) {
  const { page = 1, limit = 20, sort = { createdAt: -1 }, projection } = options;
  const skip = (page - 1) * limit;

  const [docs, total] = await Promise.all([
    this.find(filter, projection).sort(sort).skip(skip).limit(limit).lean(),
    this.countDocuments(filter)
  ]);

  return { docs, total, page, limit, pages: Math.ceil(total / limit) };
};

const Post = model('Post', postSchema);
const page = await Post.paginate({ published: true }, { page: 3, limit: 15 });
```

## Cursor-Based Pagination (Keyset)

Cursor pagination avoids the performance cost of large `skip` values and provides stable results:

```javascript
async function getNextPage(model, filter = {}, afterId = null, limit = 20) {
  const query = { ...filter };
  if (afterId) {
    query._id = { $gt: afterId }; // use ObjectId ordering
  }

  const docs = await model.find(query).sort({ _id: 1 }).limit(limit + 1).lean();
  const hasNext = docs.length > limit;
  if (hasNext) docs.pop();

  return {
    docs,
    hasNext,
    nextCursor: hasNext ? docs[docs.length - 1]._id : null
  };
}

// First page
const page1 = await getNextPage(Post, { published: true });

// Next page using cursor from previous response
const page2 = await getNextPage(Post, { published: true }, page1.nextCursor);
```

## Using the mongoose-paginate-v2 Package

```bash
npm install mongoose-paginate-v2
```

```javascript
const paginate = require('mongoose-paginate-v2');
postSchema.plugin(paginate);

const Post = model('Post', postSchema);

const result = await Post.paginate(
  { published: true },
  { page: 1, limit: 10, sort: { createdAt: -1 }, lean: true }
);
// result.docs, result.totalDocs, result.totalPages, result.nextPage
```

## Performance Tips

```text
- Always index the sort field (e.g., createdAt, _id)
- Avoid skip > 10,000 - use cursor-based pagination instead
- Use .lean() for read-only pagination endpoints
- Use countDocuments() instead of count() (deprecated)
- Cache total count for frequently-accessed pages
```

## Summary

Offset-based pagination with `skip` and `limit` is simple but degrades at large offsets. Cursor-based pagination uses a stable identifier (like `_id`) to fetch the next page efficiently without skip. Use the `mongoose-paginate-v2` plugin for a ready-made solution. Always index sort fields and prefer `.lean()` for pagination endpoints that serve large datasets.
