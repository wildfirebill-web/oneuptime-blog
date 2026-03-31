# How to Design One-to-Many Relationships in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, One-To-Many, Schema Design, Relationships

Description: Learn how to model one-to-many relationships in MongoDB using embedding arrays, referencing, and hybrid approaches with practical guidance on choosing the right strategy.

---

## One-to-Many in MongoDB

A one-to-many relationship exists when one parent document is associated with multiple child documents. Examples include a `blog post` with many `comments`, a `customer` with many `orders`, or a `product` with many `reviews`.

MongoDB supports three main approaches: embedding arrays, child-references, and parent-references.

## Approach 1: Embedding an Array

Embed child documents as an array in the parent:

```javascript
// posts collection
{
  _id: ObjectId("post001"),
  title: "Getting Started with MongoDB",
  content: "...",
  comments: [
    {
      author: "alice",
      body: "Great post!",
      createdAt: ISODate("2026-03-01T00:00:00Z")
    },
    {
      author: "bob",
      body: "Very helpful.",
      createdAt: ISODate("2026-03-02T00:00:00Z")
    }
  ]
}
```

Add a new comment:

```javascript
db.posts.updateOne(
  { _id: ObjectId("post001") },
  {
    $push: {
      comments: {
        author: "carol",
        body: "Thanks for this!",
        createdAt: new Date()
      }
    }
  }
)
```

**Best for**: small number of children (< 100), always accessed with the parent, rarely queried independently.

## Approach 2: Child-References (Parent Stores Array of Child IDs)

Store child IDs in the parent document:

```javascript
// customers collection
{
  _id: ObjectId("cust001"),
  name: "Alice Smith",
  orderIds: [ObjectId("ord001"), ObjectId("ord002"), ObjectId("ord003")]
}

// orders collection
{
  _id: ObjectId("ord001"),
  customerId: ObjectId("cust001"),
  total: 129.99,
  status: "delivered"
}
```

Fetch all orders for a customer:

```javascript
const customer = db.customers.findOne({ _id: ObjectId("cust001") });
const orders = db.orders.find({ _id: { $in: customer.orderIds } }).toArray();
```

**Best for**: moderate number of children, children queried independently, children updated frequently.

## Approach 3: Parent-Reference (Child Stores Parent ID)

Each child document stores a reference to its parent - the most common pattern in relational-style MongoDB design:

```javascript
// orders collection
{
  _id: ObjectId("ord001"),
  customerId: ObjectId("cust001"),
  total: 129.99,
  status: "delivered",
  createdAt: ISODate("2026-02-15T00:00:00Z")
}
```

Index on `customerId` for efficient lookups:

```javascript
db.orders.createIndex({ customerId: 1, createdAt: -1 })
```

Fetch customer's orders:

```javascript
db.orders.find({ customerId: ObjectId("cust001") }).sort({ createdAt: -1 })
```

Use `$lookup` to join:

```javascript
db.customers.aggregate([
  { $match: { _id: ObjectId("cust001") } },
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customerId",
      as: "orders"
    }
  }
])
```

**Best for**: large number of children (unbounded), children frequently accessed without the parent, heavy write operations on children.

## Choosing the Right Approach

```text
Scenario                              | Best Approach
--------------------------------------|---------------------------
Few children, always with parent      | Embed array
Children queried independently often  | Parent-reference
Unbounded number of children          | Parent-reference
Children updated atomically with parent | Embed array
Children shared across parents        | Parent-reference
```

## The Sixteen Megabyte Rule

MongoDB documents are limited to 16MB. If children could grow unboundedly (e.g., all comments on a viral post), embedding will eventually fail. Use parent-referencing for unbounded relationships.

## Summary

For one-to-many relationships, embed arrays when children are few, bounded, and always accessed with the parent. Use parent-referencing (child stores parent ID) for large or unbounded child sets, as it scales without document size limits and allows efficient independent queries on children. Index the foreign key field in the child collection and use `$lookup` when you need joined results in aggregations.
