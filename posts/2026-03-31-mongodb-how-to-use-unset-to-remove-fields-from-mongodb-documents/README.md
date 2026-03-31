# How to Use $unset to Remove Fields from MongoDB Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operators, $unset, Document Updates, NoSQL

Description: Learn how to use MongoDB's $unset update operator to permanently remove fields from documents, including nested fields and array elements.

---

## What Is the $unset Operator?

The `$unset` operator removes specified fields from a document. Once removed, the field no longer appears in the document. If the field does not exist, `$unset` does nothing (no error is thrown).

```javascript
db.collection.updateOne(
  { filter },
  { $unset: { fieldName: "" } }
)
```

The value assigned in `$unset` (typically `""`) is ignored - only the field name matters.

## Basic Example

Given a `users` collection:

```javascript
{ _id: 1, name: "Alice", age: 28, tempToken: "abc123", createdAt: ISODate("...") }
```

Remove the temporary token:

```javascript
db.users.updateOne(
  { _id: 1 },
  { $unset: { tempToken: "" } }
)
```

Result:

```javascript
{ _id: 1, name: "Alice", age: 28, createdAt: ISODate("...") }
```

## Removing Multiple Fields

```javascript
db.users.updateOne(
  { _id: 1 },
  { $unset: { tempToken: "", resetCode: "", legacyField: "" } }
)
```

## Removing Nested Fields

Use dot notation to target nested fields:

```javascript
// Remove only the 'middleName' from a nested 'profile' object
db.users.updateOne(
  { _id: 1 },
  { $unset: { "profile.middleName": "" } }
)
```

The parent `profile` object remains; only `middleName` is removed.

## Removing Fields from Array Elements

When `$unset` targets an array element by index, it sets that element to `null` rather than removing it (to preserve array indexing):

```javascript
// Sets scores[1] to null - does NOT remove the element
db.students.updateOne(
  { _id: 1 },
  { $unset: { "scores.1": "" } }
)
// Before: { scores: [85, 90, 78] }
// After:  { scores: [85, null, 78] }
```

To remove null values from an array after unsetting, use `$pull`:

```javascript
db.students.updateOne(
  { _id: 1 },
  { $pull: { scores: null } }
)
```

## Removing Fields from Many Documents

Use `updateMany` with a filter to remove a deprecated field from all documents:

```javascript
db.products.updateMany(
  {},
  { $unset: { legacyCategory: "", oldPrice: "" } }
)
```

## Removing Fields That May Not Exist

`$unset` is safe to use on fields that might not exist - it simply does nothing:

```javascript
db.users.updateMany(
  {},
  { $unset: { deprecatedField: "" } }
)
```

No error occurs even if some documents don't have `deprecatedField`.

## Schema Migration Use Case

When migrating a schema, `$unset` is invaluable for cleaning up old fields:

```javascript
// Step 1: Migrate data from old field to new field
db.orders.updateMany(
  { oldStatus: { $exists: true } },
  [
    { $set: { status: "$oldStatus" } },
    { $unset: "oldStatus" }
  ]
)
```

Note: In aggregation pipeline updates (using array syntax), `$unset` accepts a string or array of strings directly.

## Aggregation Pipeline Syntax

Within pipeline-style updates, `$unset` syntax is slightly different:

```javascript
db.users.updateMany(
  {},
  [
    { $unset: ["tempToken", "resetCode"] }
  ]
)
```

## Combining $set and $unset

```javascript
db.users.updateOne(
  { _id: 1 },
  {
    $set: { updatedAt: new Date(), newField: "value" },
    $unset: { oldField: "", anotherOld: "" }
  }
)
```

## Summary

The `$unset` operator cleanly removes fields from MongoDB documents without affecting other fields. It handles missing fields gracefully, supports dot notation for nested paths, and works in both standard and aggregation pipeline update styles. Use it for schema migrations, removing temporary tokens, and cleaning up deprecated fields at scale.
