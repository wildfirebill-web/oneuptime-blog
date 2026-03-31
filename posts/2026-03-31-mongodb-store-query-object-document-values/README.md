# How to Store and Query Object/Document Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Document, Embedded, Query, Schema

Description: Understand how MongoDB stores embedded documents (objects), how to query nested fields with dot notation, and when to embed vs reference.

---

## Embedded Documents in MongoDB

MongoDB is a document database, and every document can contain nested documents (sub-documents) as field values. These are stored as BSON type `object` (type 3) and are the primary mechanism for representing one-to-one or one-to-few relationships without joins.

## Inserting Embedded Documents

```javascript
db.users.insertMany([
  {
    name: "Alice",
    address: {
      street: "123 Main St",
      city: "Springfield",
      state: "IL",
      zip: "62701",
    },
    preferences: {
      theme: "dark",
      notifications: true,
    },
  },
  {
    name: "Bob",
    address: {
      street: "456 Oak Ave",
      city: "Shelbyville",
      state: "IL",
      zip: "62565",
    },
    preferences: {
      theme: "light",
      notifications: false,
    },
  },
]);
```

## Querying Nested Fields with Dot Notation

```javascript
// Query a specific nested field
db.users.find({ "address.city": "Springfield" });

// Query multiple nested conditions
db.users.find({ "address.state": "IL", "preferences.theme": "dark" });
```

## Exact Object Match

Querying an entire embedded document with exact object equality requires the document to match exactly - including field order:

```javascript
// This only matches if address is EXACTLY this object in this field order
db.users.find({
  address: {
    street: "123 Main St",
    city: "Springfield",
    state: "IL",
    zip: "62701",
  },
});
```

Use dot notation instead to query individual fields without requiring exact object equality.

## Updating Nested Fields

```javascript
// Update a single nested field without touching the rest
db.users.updateOne(
  { name: "Alice" },
  { $set: { "address.city": "Chicago", "preferences.theme": "light" } }
);

// Add a new field to an embedded document
db.users.updateOne({ name: "Alice" }, { $set: { "address.country": "US" } });
```

## Aggregation on Nested Fields

```javascript
db.users.aggregate([
  {
    $group: {
      _id: "$address.state",
      userCount: { $sum: 1 },
    },
  },
  { $sort: { userCount: -1 } },
]);
```

Extract nested fields with `$project`:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      city: "$address.city",
      state: "$address.state",
    },
  },
]);
```

## Indexing Nested Fields

Create indexes on nested fields using dot notation:

```javascript
db.users.createIndex({ "address.city": 1, "address.state": 1 });

// Verify it is used
db.users.find({ "address.city": "Springfield" }).explain("executionStats");
```

## Embed vs Reference Decision

Embed when:
- Data is always accessed together with the parent document
- The nested document belongs exclusively to one parent
- The nested data set is bounded in size (does not grow indefinitely)

Reference (store an ObjectId and use `$lookup`) when:
- The nested entity is shared across many parent documents
- The nested data grows unboundedly (e.g., order line items)
- You need to query the nested entity independently

## Schema Validation for Embedded Documents

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "address"],
      properties: {
        address: {
          bsonType: "object",
          required: ["city", "state"],
          properties: {
            city: { bsonType: "string" },
            state: { bsonType: "string", minLength: 2, maxLength: 2 },
          },
        },
      },
    },
  },
});
```

## Summary

MongoDB embedded documents store related data within a single document using BSON object type, eliminating join operations for frequently co-accessed data. Query nested fields with dot notation for flexibility, as exact object matching requires precise field ordering. Index nested fields using dot notation syntax. Choose embedding for exclusive, bounded, co-accessed relationships and referencing for shared, unbounded, or independently queried entities.
