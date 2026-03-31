# How to Store and Query Boolean Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Boolean, Query, Schema, BSON

Description: Learn how MongoDB stores boolean values in BSON, how to query them efficiently, and common pitfalls around truthy values and type coercion.

---

## Boolean Type in BSON

MongoDB stores booleans as BSON type `bool` (type number 8). Unlike JavaScript's loose equality, MongoDB queries perform strict type matching, so `true` (boolean) and `1` (integer) are not interchangeable in queries.

## Inserting Boolean Values

```javascript
db.users.insertMany([
  { name: "Alice", isActive: true, isAdmin: false },
  { name: "Bob", isActive: true, isAdmin: true },
  { name: "Carol", isActive: false, isAdmin: false },
]);
```

## Basic Boolean Queries

```javascript
// Find all active users
db.users.find({ isActive: true });

// Find all non-admin users
db.users.find({ isAdmin: false });

// Find active admins
db.users.find({ isActive: true, isAdmin: true });
```

## Querying for the Absence of a Boolean Field

When a field might not exist on all documents, distinguish between `false` and `null`/missing:

```javascript
// Returns documents where isActive is false OR does not exist
db.users.find({ isActive: { $ne: true } });

// Returns ONLY documents where isActive explicitly equals false
db.users.find({ isActive: false });

// Returns documents where the field does not exist at all
db.users.find({ isActive: { $exists: false } });
```

## Using $type to Filter Strictly by Boolean

If your collection has mixed data (some documents store `1`/`0` as integers instead of booleans), use `$type` to find only true boolean fields:

```javascript
// BSON type 8 is bool
db.users.find({ isActive: { $type: "bool" } });
```

## Converting Truthy Integers to Booleans

If legacy data stores `1` for true and `0` for false, use an aggregation to convert:

```javascript
db.users.aggregate([
  {
    $addFields: {
      isActive: { $toBool: "$isActive" },
    },
  },
]);
```

Then update in place:

```javascript
db.users.updateMany({}, [
  { $set: { isActive: { $toBool: "$isActive" } } },
]);
```

## Indexing Boolean Fields

Boolean fields have very low cardinality (only two values), so a standalone index provides little benefit. Instead, combine the boolean field with a higher-cardinality field in a compound index:

```javascript
// Effective: compound index where boolean acts as a filter prefix
db.users.createIndex({ isActive: 1, createdAt: -1 });
```

For queries that always filter `isActive: true`, place it first in the index to narrow the range quickly.

## Negation Queries

Negation queries (`$ne`, `$not`) cannot efficiently use a standard index and often result in a collection scan. Prefer querying the affirmative value:

```javascript
// Less efficient - cannot use index
db.users.find({ isAdmin: { $ne: true } });

// More efficient - reframe if your query intent allows
db.users.find({ isAdmin: false });
```

## Schema Validation

Enforce that a boolean field is always a boolean (not an integer or null) using JSON Schema validation:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        isActive: { bsonType: "bool" },
        isAdmin: { bsonType: "bool" },
      },
    },
  },
});
```

## Summary

MongoDB stores boolean values as BSON type 8, distinct from integers. Use explicit `true`/`false` in queries rather than `1`/`0` to avoid type mismatch. For low-cardinality boolean fields, compound indexes outperform standalone boolean indexes. Use `$exists` and `$ne` carefully to distinguish between `false` and missing fields, and enforce boolean types with JSON Schema validation to prevent accidental integer storage.
