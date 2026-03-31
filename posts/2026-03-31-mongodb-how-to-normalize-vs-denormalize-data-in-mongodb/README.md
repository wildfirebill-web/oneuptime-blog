# How to Normalize vs Denormalize Data in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Normalization, Denormalization, Performance

Description: Understand when to embed documents versus reference them in MongoDB, and how each choice affects query performance and data consistency.

---

## The Core Trade-off

In relational databases, normalization is the default and joins are cheap. In MongoDB, joins via `$lookup` exist but are more expensive than in relational databases. The decision to embed (denormalize) or reference (normalize) data shapes query performance, consistency, and write amplification.

## When to Embed (Denormalize)

Embedding is appropriate when data is read together, owned by a single parent, and does not need to be updated independently at high frequency.

```javascript
// Embedded address in a user document - good for profile pages
{
  _id: ObjectId(),
  name: "Alice Smith",
  email: "alice@example.com",
  address: {
    street: "123 Main St",
    city: "Austin",
    state: "TX",
    zip: "78701"
  }
}
```

Reading a user profile fetches the address in a single document read. No additional query needed.

## When to Reference (Normalize)

Reference when data is shared across many documents, updated frequently, or unbounded in size.

```javascript
// Author referenced from many posts - normalize to avoid duplication
{
  _id: ObjectId("post1"),
  title: "Getting Started with MongoDB",
  authorId: ObjectId("author1"),
  publishedAt: new Date()
}

// Author document
{
  _id: ObjectId("author1"),
  name: "Bob Jones",
  bio: "Senior engineer at Acme Corp",
  avatarUrl: "https://cdn.example.com/bob.jpg"
}
```

When the author updates their bio, only one document changes.

## The Hybrid Pattern

Many real-world schemas use a hybrid: embed a small snapshot for display, store the authoritative data separately.

```javascript
{
  _id: ObjectId("post1"),
  title: "Getting Started with MongoDB",
  author: {
    _id: ObjectId("author1"),
    name: "Bob Jones",        // denormalized for display
    avatarUrl: "https://..."  // denormalized for display
  }
}
```

This trades storage for read speed. When an author's name changes, you must update embedded copies.

## Using $lookup for Normalized Data

When you need full author details alongside posts, use the aggregation pipeline:

```javascript
db.posts.aggregate([
  { $match: { publishedAt: { $gte: new Date("2025-01-01") } } },
  {
    $lookup: {
      from: "authors",
      localField: "authorId",
      foreignField: "_id",
      as: "author"
    }
  },
  { $unwind: "$author" }
]);
```

## Sizing Guidance

Embed when the subdocument is stable and the array or object has a bounded, small size - typically fewer than 100 elements. Reference when arrays can grow without bound, as MongoDB has a 16MB document size limit.

```javascript
// Risky - tags array could grow without bound over time
{
  _id: ObjectId(),
  productId: "SKU-001",
  tags: ["electronics", "sale", "popular", /* ... could be thousands */]
}

// Better - reference tags separately if unbounded
{
  _id: ObjectId(),
  productId: "SKU-001",
  tagIds: [ObjectId("tag1"), ObjectId("tag2")]
}
```

## Summary

Embedding (denormalization) in MongoDB trades write complexity for fast single-document reads and is ideal for data owned by one parent with bounded size. Referencing (normalization) suits shared or frequently updated data at the cost of requiring `$lookup` joins. Most production schemas use a hybrid, embedding stable display data while referencing authoritative sources.
