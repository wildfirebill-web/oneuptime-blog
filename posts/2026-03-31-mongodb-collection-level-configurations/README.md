# How to Handle Collection-Level Configurations in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Configuration, Schema, Collation

Description: Configure MongoDB collections with schema validation, collation, capped limits, and read/write concerns to enforce data quality at the collection level.

---

## Overview

MongoDB collections support a range of configuration options that go beyond simple document storage. Schema validation enforces data integrity, collation controls sorting behavior for different languages, capped collections set fixed storage limits, and collection-level read/write concerns override cluster defaults. Setting these configurations at the collection level ensures consistent behavior regardless of which application code writes to them.

## Schema Validation

Enforce required fields and data types at the collection level using JSON Schema validation.

```javascript
await db.createCollection("invoices", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "amount", "currency", "status", "createdAt"],
      properties: {
        customerId: { bsonType: "string", minLength: 1 },
        amount: { bsonType: "decimal", minimum: 0 },
        currency: { bsonType: "string", enum: ["USD", "EUR", "GBP", "JPY"] },
        status: {
          bsonType: "string",
          enum: ["draft", "sent", "paid", "overdue", "cancelled"],
        },
        createdAt: { bsonType: "date" },
        lineItems: {
          bsonType: "array",
          minItems: 1,
          items: {
            bsonType: "object",
            required: ["description", "qty", "unitPrice"],
            properties: {
              description: { bsonType: "string" },
              qty: { bsonType: "int", minimum: 1 },
              unitPrice: { bsonType: "decimal", minimum: 0 },
            },
          },
        },
      },
    },
  },
  validationLevel: "strict",
  validationAction: "error",
});
```

## Collation for Language-Aware Sorting

Configure a collection's default collation so text sorting follows locale-specific rules.

```javascript
await db.createCollection("products", {
  collation: {
    locale: "fr",
    strength: 2,
    caseLevel: false,
  },
});

// Queries inherit the collection collation automatically
await db.collection("products").find({}).sort({ name: 1 }).toArray();

// Override collation for a specific query
await db.collection("products").find({}).sort({ name: 1 })
  .collation({ locale: "en", strength: 1 })
  .toArray();
```

## Capped Collection Configuration

Create a capped collection with a fixed maximum size and document count for log buffers or rolling windows.

```javascript
await db.createCollection("app_logs", {
  capped: true,
  size: 104857600,  // 100 MB
  max: 1000000,     // max 1 million documents
});
```

## Modifying Validation on Existing Collections

Update the validator for an existing collection without dropping it.

```javascript
await db.command({
  collMod: "invoices",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "amount", "currency", "status", "createdAt"],
      // ... updated schema
    },
  },
  validationLevel: "moderate",
  validationAction: "warn",
});
```

Use `validationAction: "warn"` when rolling out a new validation rule on an existing collection with legacy data.

## Listing Collection Configurations

Inspect current options for all collections in a database.

```javascript
const collections = await db.listCollections().toArray();
for (const col of collections) {
  console.log(`${col.name}:`, JSON.stringify({
    validator: col.options?.validator ? "yes" : "no",
    capped: col.options?.capped || false,
    collation: col.options?.collation?.locale || "default",
  }));
}
```

## Summary

MongoDB collection-level configurations let you enforce data integrity through schema validation, control text sorting with collation settings, cap storage usage for log-style collections, and tune validation strictness without changing application code. Always define these configurations when creating collections, and use `collMod` to update them as requirements evolve.
