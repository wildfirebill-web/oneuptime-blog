# How to Use DataLoaders for Efficient MongoDB Queries in GraphQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GraphQL, DataLoader, Performance, Apollo

Description: Learn how to use DataLoader to batch and cache MongoDB queries in GraphQL resolvers, eliminating the N+1 query problem with Mongoose.

---

The N+1 query problem is the most common performance pitfall in GraphQL APIs. When resolving a list of posts each with an author, a naive implementation fires one MongoDB query per post. DataLoader solves this by batching all IDs into a single query per request.

## Installing DataLoader

```bash
npm install dataloader
```

## Understanding the N+1 Problem

Without DataLoader, resolving 10 posts with their authors fires 11 MongoDB queries:

```javascript
// BAD - N+1 problem
const resolvers = {
  Post: {
    author: async (post) => {
      // This fires once per post in the list!
      return User.findById(post.authorId);
    },
  },
};
```

## Creating DataLoaders

DataLoader batches calls that happen in the same event loop tick:

```javascript
const DataLoader = require('dataloader');
const User = require('./models/User');
const Tag = require('./models/Tag');

function createUserLoader() {
  return new DataLoader(async (userIds) => {
    // One query for all IDs
    const users = await User.find({ _id: { $in: userIds } }).lean();

    // Map results back to the original order
    const userMap = new Map(users.map((u) => [u._id.toString(), u]));
    return userIds.map((id) => userMap.get(id.toString()) || null);
  });
}

function createTagLoader() {
  return new DataLoader(async (postIds) => {
    // Batch load tags grouped by postId
    const tags = await Tag.find({ postId: { $in: postIds } }).lean();

    const tagMap = new Map();
    for (const tag of tags) {
      const key = tag.postId.toString();
      if (!tagMap.has(key)) tagMap.set(key, []);
      tagMap.get(key).push(tag);
    }

    return postIds.map((id) => tagMap.get(id.toString()) || []);
  });
}
```

## Adding DataLoaders to Apollo Context

Create fresh loaders per request to avoid cache pollution between requests:

```javascript
const { startStandaloneServer } = require('@apollo/server/standalone');

await startStandaloneServer(server, {
  context: async ({ req }) => {
    return {
      loaders: {
        user: createUserLoader(),
        tag: createTagLoader(),
      },
    };
  },
});
```

## Using DataLoaders in Resolvers

```javascript
const resolvers = {
  Post: {
    // Now uses DataLoader - batches into ONE query regardless of list size
    author: async (post, _, { loaders }) => {
      return loaders.user.load(post.authorId.toString());
    },

    relatedTags: async (post, _, { loaders }) => {
      return loaders.tag.load(post._id.toString());
    },
  },
};
```

## Batch Function Ordering Contract

The batch function must return results in the same order as the input IDs. This is critical:

```javascript
function createProductLoader() {
  return new DataLoader(async (productIds) => {
    const products = await Product.find({ _id: { $in: productIds } }).lean();
    const productMap = new Map(
      products.map((p) => [p._id.toString(), p])
    );

    // IMPORTANT: preserve input order, return null for missing IDs
    return productIds.map((id) => productMap.get(id.toString()) ?? null);
  });
}
```

## Priming the Cache

Pre-populate the loader cache with data you already have:

```javascript
const resolvers = {
  Query: {
    posts: async (_, { limit }, { loaders }) => {
      const posts = await Post.find().populate('authorId').limit(limit).lean();

      // Prime the user loader so nested author resolutions hit cache
      for (const post of posts) {
        if (post.authorId) {
          loaders.user.prime(post.authorId._id.toString(), post.authorId);
        }
      }

      return posts;
    },
  },
};
```

## Measuring the Improvement

With DataLoader on a list of 100 posts:

```text
Without DataLoader: 101 queries (1 list + 100 author lookups)
With DataLoader:      2 queries (1 list + 1 batched author lookup)
```

## Summary

DataLoader eliminates the N+1 problem by collecting all IDs requested in the same event loop tick and issuing a single batched MongoDB query. Always create new DataLoader instances per request to keep the per-request cache isolated. Use loader priming to further reduce redundant queries when you already have the data in a parent resolver.
