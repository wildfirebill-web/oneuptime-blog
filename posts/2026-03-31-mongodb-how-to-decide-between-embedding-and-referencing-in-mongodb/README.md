# How to Decide Between Embedding and Referencing in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Embedding, Referencing, Schema Design

Description: A practical guide to deciding when to embed documents versus reference them in MongoDB, with decision criteria and real-world examples.

---

## The Core Trade-off

MongoDB's flexible document model lets you either embed related data within a document or store it in separate collections and link via references. The right choice depends on your data's access patterns, size, and update frequency.

## Embedding: What It Means

Embedding means storing related data as a nested subdocument or array within the parent document:

```javascript
{
  _id: ObjectId("order001"),
  customerId: "cust123",
  items: [
    { productId: "prod001", name: "Laptop", qty: 1, price: 999.99 },
    { productId: "prod002", name: "Mouse", qty: 2, price: 29.99 }
  ],
  shippingAddress: {
    street: "123 Main St",
    city: "San Francisco",
    state: "CA",
    zip: "94102"
  }
}
```

## Referencing: What It Means

Referencing stores a reference (usually `ObjectId`) to a document in another collection:

```javascript
// Order document
{
  _id: ObjectId("order001"),
  customerId: ObjectId("cust123"),
  productIds: [ObjectId("prod001"), ObjectId("prod002")]
}

// Customer document (separate collection)
{
  _id: ObjectId("cust123"),
  name: "Alice Smith",
  email: "alice@example.com"
}
```

## Decision Criteria

### 1. Access Pattern

If you always need the related data when you load the parent, embed it. If you often need the data independently, use references:

```text
"Show order with all its items"           -> Embed items
"Show all orders for a customer"          -> Reference (customerId in orders)
"Show product details for all orders"     -> Reference (product data changes)
```

### 2. Cardinality and Boundedness

```javascript
// One-to-few (embed): user has a few addresses
{
  _id: ObjectId("u001"),
  addresses: [
    { type: "home", street: "123 Main St", city: "SF" },
    { type: "work", street: "456 Market St", city: "SF" }
  ]
}

// One-to-many (reference): customer has thousands of orders
// orders collection: { customerId: ObjectId("u001"), ... }
db.orders.createIndex({ customerId: 1 })
```

### 3. Update Frequency

If embedded data changes often and independently, referencing avoids scattered updates:

```javascript
// Product prices change frequently - reference is better
// Otherwise you'd update every order that contains the product

// orders store only productId, not the full product document
{
  items: [
    { productId: ObjectId("p001"), qty: 2, priceAtPurchase: 29.99 }
  ]
}
```

Note: `priceAtPurchase` is snapshot data (correct at order time) - this is intentional.

### 4. Atomic Operations

Embedding allows atomic updates of parent and related data in a single operation:

```javascript
// Atomically add item and update total
db.orders.updateOne(
  { _id: ObjectId("order001") },
  {
    $push: { items: { productId: "prod003", name: "Keyboard", qty: 1, price: 79.99 } },
    $inc: { total: 79.99 }
  }
)
```

With references, you need multi-document transactions for atomicity.

### 5. Document Size Limit

MongoDB enforces a 16MB document limit. If embedded arrays can grow unboundedly (comments, events, logs), use references:

```javascript
// Risk: a popular post could accumulate millions of comments
// DON'T embed unbounded data:
{ _id: "post001", comments: [ /* millions of items */ ] }

// DO reference:
// comments collection: { postId: "post001", body: "...", createdAt: ... }
db.comments.createIndex({ postId: 1, createdAt: -1 })
```

## Quick Decision Framework

```text
Question                                    | Answer   | Action
--------------------------------------------|----------|----------
Always fetched together?                    | Yes      | Embed
Updated atomically with parent?             | Yes      | Embed
Small, bounded number of children (<= 20)?  | Yes      | Embed
Children queried/updated independently?     | Yes      | Reference
Unbounded growth possible?                  | Yes      | Reference
Data shared across multiple parents?        | Yes      | Reference
Children have their own relationships?      | Yes      | Reference
```

## Hybrid Approach: Extended Reference Pattern

Embed a subset of frequently accessed reference data to avoid lookups, while keeping the full document separate:

```javascript
// Embed only what you display in a list view
{
  _id: ObjectId("order001"),
  customer: {
    _id: ObjectId("cust123"),
    name: "Alice Smith",       // denormalized subset
    avatarUrl: "..."           // denormalized subset
  }
}
```

Update embedded snapshots when source data changes using background jobs or change streams.

## Summary

Embed related data when it is always accessed with the parent, has a small and bounded count, and is updated atomically. Use references when data is queried independently, may grow unboundedly, is shared across multiple parents, or changes frequently. The key question is always: "What does my application read and write together?" - let access patterns drive your schema design.
