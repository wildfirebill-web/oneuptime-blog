# What Is the Difference Between Embedded Documents and References in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Document, Reference, Performance

Description: Embedding nests related data inside a document for fast reads while referencing stores related data separately - choosing between them defines your schema's performance characteristics.

---

## Overview

MongoDB's flexible document model lets you represent relationships in two ways: embedding related data directly inside a document, or storing related data in a separate collection and linking via a reference. This is one of the most important schema design decisions you will make.

## Embedded Documents

Embedding places related data as a subdocument or array inside the parent document. All data is stored together in one document.

```javascript
// Embedded address inside a user document
db.users.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6789abc0001"),
  name: "Alice Johnson",
  email: "alice@example.com",
  address: {
    street: "123 Main St",
    city: "Austin",
    state: "TX",
    zip: "78701"
  },
  tags: ["premium", "newsletter"]
})
```

### Reading Embedded Data

```javascript
// Single query retrieves both user and address
const user = db.users.findOne({ email: "alice@example.com" })
console.log(user.address.city) // "Austin"
```

No additional query is needed. This is the primary advantage of embedding.

## References (Manual References and DBRef)

References store related data in a separate collection and link documents using an ObjectId or other identifier.

```javascript
// Separate collections for orders and customers
db.customers.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6789abc0010"),
  name: "Bob Smith",
  email: "bob@example.com"
})

db.orders.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6789abc0020"),
  customerId: ObjectId("64a1b2c3d4e5f6789abc0010"),
  items: [{ sku: "WIDGET-A", qty: 3, price: 9.99 }],
  total: 29.97,
  createdAt: new Date()
})
```

### Reading Referenced Data

To retrieve both the order and its customer, use `$lookup`:

```javascript
db.orders.aggregate([
  { $match: { _id: ObjectId("64a1b2c3d4e5f6789abc0020") } },
  { $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }},
  { $unwind: "$customer" }
])
```

This requires an aggregation stage where embedding would need only a `findOne()`.

## When to Embed

Embedding is the right choice when:

- The relationship is "owns" or "contains" (one-to-one or one-to-few)
- The embedded data is always read alongside the parent
- The embedded data does not grow unbounded
- You want atomicity - updates to the parent and its embedded data are a single operation

```javascript
// Good candidate for embedding: product with limited variant data
db.products.insertOne({
  name: "T-Shirt",
  variants: [
    { size: "S", color: "red", stock: 12 },
    { size: "M", color: "red", stock: 5 },
    { size: "L", color: "blue", stock: 8 }
  ]
})
```

## When to Use References

References are the right choice when:

- The related data grows without bound (one-to-many with many entries)
- The related entity is accessed independently
- Multiple parents reference the same child (many-to-many)
- Documents would exceed the 16 MB BSON limit if embedded

```javascript
// Good candidate for references: blog post comments (can be thousands)
db.comments.insertOne({
  postId: ObjectId("64a1b2c3d4e5f6789abc0030"),
  author: "Carol",
  body: "Great article!",
  createdAt: new Date()
})
```

## Hybrid Approach

Many schemas use a hybrid pattern - embed a summary for fast reads and reference the full entity for detail pages.

```javascript
// Order with embedded customer summary + full customer reference
db.orders.insertOne({
  customerId: ObjectId("64a1b2c3d4e5f6789abc0010"),
  customerName: "Bob Smith",   // Embedded for display without $lookup
  total: 29.97,
  items: [{ sku: "WIDGET-A", qty: 3 }]
})
```

## Key Differences

| Factor | Embedding | References |
|---|---|---|
| Read performance | Faster (single query) | Slower ($lookup required) |
| Write atomicity | Atomic by default | Requires transactions for multi-doc |
| Document size risk | Can hit 16 MB limit | No size concern |
| Data duplication | Yes (denormalized) | No (normalized) |
| Independent access | Harder | Easy |
| Unbounded growth | Dangerous | Safe |

## Summary

Embedding optimizes for read performance by co-locating related data in a single document, making it ideal for contained, bounded relationships. References normalize data across collections and suit relationships where the child data grows without bound or is accessed independently. Most real-world MongoDB schemas use a combination: embed what is always read together and reference what is large, shared, or independent. Start with embedding, then extract to references when documents approach size limits or when the embedded array becomes too large to be practical.
