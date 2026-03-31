# How to Index for Nested Document Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Nested Document, Dot Notation, Query

Description: Learn how to create and use indexes on nested document fields using dot notation, and understand the difference between indexing specific fields vs entire subdocuments.

---

## Indexing Nested Fields with Dot Notation

MongoDB lets you create indexes on fields within embedded documents using dot notation. This is the correct and efficient approach for querying nested data.

```javascript
db.users.insertMany([
  {
    name: "Alice",
    address: { city: "Chicago", state: "IL", zip: "60601" },
    contact: { email: "alice@example.com", phone: "555-1234" },
  },
  {
    name: "Bob",
    address: { city: "Springfield", state: "IL", zip: "62701" },
    contact: { email: "bob@example.com", phone: "555-5678" },
  },
]);

// Create an index on a nested field
db.users.createIndex({ "address.city": 1 });
db.users.createIndex({ "contact.email": 1 }, { unique: true });
```

## Querying Nested Fields

```javascript
// Efficient - uses the index on address.city
db.users.find({ "address.city": "Chicago" });

// Compound query on nested fields
db.users.find({ "address.state": "IL", "address.city": "Springfield" });
```

## Compound Index on Nested Fields

```javascript
// Compound index covering state + city for filtered city lookups
db.users.createIndex({ "address.state": 1, "address.city": 1 });

// This query uses the compound index efficiently
db.users.find({ "address.state": "IL", "address.city": "Chicago" });
```

## Exact Subdocument Matching (Avoid This)

Indexing the entire subdocument and querying with exact object equality is fragile because field order matters:

```javascript
// Index on the entire address subdocument
db.users.createIndex({ address: 1 });

// This ONLY matches if address is EXACTLY this object in this exact field order
db.users.find({ address: { city: "Chicago", state: "IL", zip: "60601" } });
// Will NOT match { state: "IL", city: "Chicago", zip: "60601" }
```

Always prefer indexing specific nested fields with dot notation over indexing entire subdocuments.

## Wildcard Index for Dynamic Nested Fields

When nested fields are dynamic (e.g., user-defined metadata), use a wildcard index:

```javascript
db.products.insertMany([
  { name: "Widget", attributes: { color: "red", size: "M" } },
  { name: "Gadget", attributes: { material: "aluminum", weight: "200g" } },
]);

// Wildcard index covers all nested fields under attributes
db.products.createIndex({ "attributes.$**": 1 });

// Both queries can use the wildcard index
db.products.find({ "attributes.color": "red" });
db.products.find({ "attributes.material": "aluminum" });
```

Wildcard indexes have higher write overhead since every field generates an index entry.

## Verify Nested Field Index Usage

```javascript
db.users.find({ "address.city": "Chicago", "address.state": "IL" })
  .explain("executionStats");
```

Expected output confirming index use:

```json
{
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "address.state_1_address.city_1",
      "isMultiKey": false
    }
  }
}
```

`isMultiKey: false` confirms the indexed field is not an array.

## Deeply Nested Fields

Dot notation works to any depth:

```javascript
db.orders.createIndex({ "shipping.address.city": 1 });
db.orders.find({ "shipping.address.city": "Chicago" });
```

## Summary

Index nested document fields using dot notation (`"address.city": 1`) rather than indexing the entire subdocument to avoid field-order dependency in queries. Create compound indexes on multiple nested fields for multi-field filters. Use wildcard indexes (`"attributes.$**": 1`) for collections with dynamic or unknown nested field names. Always verify nested index usage with `explain()` and confirm `IXSCAN` is the winning plan.
