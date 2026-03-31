# How to Implement Pagination in GraphQL with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, GraphQL, Pagination, Cursor, Apollo

Description: Learn how to implement cursor-based and offset-based pagination in a GraphQL API backed by MongoDB, with proper connection types and page info.

---

Pagination in GraphQL APIs backed by MongoDB is commonly implemented in two styles: offset pagination (simple but limited) and cursor-based pagination (the Relay Connection spec). This guide covers both approaches with working Mongoose examples.

## Offset Pagination

Offset pagination is simple to implement but can return duplicate or skipped documents when data changes between pages.

### Schema

```graphql
type PostPage {
  items: [Post!]!
  total: Int!
  hasMore: Boolean!
}

type Query {
  postsOffset(limit: Int!, offset: Int!): PostPage!
}
```

### Resolver

```javascript
const resolvers = {
  Query: {
    postsOffset: async (_, { limit, offset }) => {
      const [items, total] = await Promise.all([
        Post.find({ published: true })
          .sort({ createdAt: -1 })
          .skip(offset)
          .limit(limit)
          .lean(),
        Post.countDocuments({ published: true }),
      ]);

      return { items, total, hasMore: offset + items.length < total };
    },
  },
};
```

## Cursor-Based Pagination (Relay Connection)

Cursor-based pagination is more reliable for real-time data. The cursor encodes the position in the result set.

### Schema

```graphql
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type Query {
  postsConnection(first: Int, after: String): PostConnection!
}
```

### Cursor Encoding

Encode the MongoDB `_id` as a base64 cursor:

```javascript
function encodeCursor(id) {
  return Buffer.from(id.toString()).toString('base64');
}

function decodeCursor(cursor) {
  return Buffer.from(cursor, 'base64').toString('utf8');
}
```

### Resolver

```javascript
const { ObjectId } = require('mongoose').Types;

const resolvers = {
  Query: {
    postsConnection: async (_, { first = 20, after }) => {
      const query = { published: true };

      if (after) {
        const decodedId = decodeCursor(after);
        query._id = { $lt: new ObjectId(decodedId) };
      }

      const posts = await Post.find(query)
        .sort({ _id: -1 })
        .limit(first + 1) // fetch one extra to check hasNextPage
        .lean();

      const hasNextPage = posts.length > first;
      const edges = posts.slice(0, first).map((post) => ({
        node: post,
        cursor: encodeCursor(post._id),
      }));

      const totalCount = await Post.countDocuments({ published: true });

      return {
        edges,
        pageInfo: {
          hasNextPage,
          hasPreviousPage: !!after,
          startCursor: edges[0]?.cursor ?? null,
          endCursor: edges[edges.length - 1]?.cursor ?? null,
        },
        totalCount,
      };
    },
  },
};
```

## Example GraphQL Query

```graphql
# First page
query {
  postsConnection(first: 10) {
    edges {
      cursor
      node {
        id
        title
        createdAt
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Next page using endCursor
query {
  postsConnection(first: 10, after: "NjM0YWJjZGVmMTIz") {
    edges {
      node { id title }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

## Adding a Compound Cursor for Non-ID Sort Fields

When sorting by `createdAt` instead of `_id`, encode both fields in the cursor:

```javascript
function encodeTimeCursor(post) {
  const data = JSON.stringify({ id: post._id.toString(), ts: post.createdAt.getTime() });
  return Buffer.from(data).toString('base64');
}

function decodeTimeCursor(cursor) {
  return JSON.parse(Buffer.from(cursor, 'base64').toString('utf8'));
}
```

## Summary

Use cursor-based pagination for production GraphQL APIs backed by MongoDB - it handles concurrent inserts and deletes gracefully. Use `_id`-based cursors when sorting by insertion order, and compound cursors when sorting by other fields. The Relay Connection spec gives frontend clients a predictable, consistent pagination interface.
