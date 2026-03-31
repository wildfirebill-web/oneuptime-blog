# How to Use $unset to Remove Fields from MongoDB Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $unset, Update Operator, Field Removal, Write Operation

Description: Learn how to use MongoDB's $unset operator to delete specific fields from documents, including nested fields and array elements.

---

## What Is $unset

The `$unset` operator removes a field from a document entirely. The field and its value are deleted - the field key no longer appears in the document at all. This is different from setting a field to `null`.

Syntax:

```javascript
{ $unset: { field1: "", field2: "" } }
```

The value (`""`) provided to `$unset` is irrelevant - MongoDB ignores it. Convention is to use `""` or `1`.

## Basic Examples

```javascript
// Remove the "tempPassword" and "resetToken" fields
db.users.updateOne(
  { _id: userId },
  { $unset: { tempPassword: "", resetToken: "" } }
)

// Remove deprecated field from all documents
await db.collection("products").updateMany(
  {},
  { $unset: { legacyCode: "" } }
)
```

## Removing Nested Fields

Use dot notation to remove a field inside an embedded document:

```javascript
// Remove just the "internal" notes field from the address
db.users.updateOne(
  { _id: userId },
  { $unset: { "address.internalNotes": "" } }
)
```

The rest of the `address` subdocument is preserved.

## $unset vs Setting to null

```javascript
// $unset: field is completely removed from the document
db.users.updateOne({ _id: id }, { $unset: { phone: "" } })
// Result: { _id: ..., name: "Alice", email: "..." }  (no phone key)

// $set: null keeps the field with a null value
db.users.updateOne({ _id: id }, { $set: { phone: null } })
// Result: { _id: ..., name: "Alice", email: "...", phone: null }
```

Use `$unset` when you want the field gone entirely. Use `$set: null` when you want to record that the value is explicitly absent.

## $unset on Array Elements

When used on an array element by index, `$unset` replaces the element with `null` rather than removing it (to preserve array indexes):

```javascript
// Sets the third element to null (does not shorten the array)
db.orders.updateOne(
  { _id: orderId },
  { $unset: { "items.2": "" } }
)
```

To remove elements from an array, use `$pull` instead.

## Removing Fields During a Schema Migration

```javascript
// Schema migration: remove old fields, add new ones
await db.collection("users").updateMany(
  {},
  {
    $unset: { oldField1: "", oldField2: "", deprecatedSetting: "" },
    $set: { newField: "defaultValue", updatedAt: new Date() }
  }
)
```

Combining `$unset` and `$set` in one operation is atomic.

## Checking Which Documents Have a Field Before Unsetting

```javascript
const count = await db.collection("users").countDocuments({
  tempPassword: { $exists: true }
});
console.log(`${count} documents have tempPassword field`);

await db.collection("users").updateMany(
  { tempPassword: { $exists: true } },
  { $unset: { tempPassword: "" } }
);
```

## $unset in bulkWrite()

```javascript
await db.collection("sessions").bulkWrite([
  {
    updateMany: {
      filter: { expiresAt: { $lt: new Date() } },
      update: { $unset: { token: "", refreshToken: "" } }
    }
  }
]);
```

## Common Mistakes

- Thinking `$unset: { field: null }` deletes the field - the value is irrelevant; `$unset` always removes.
- Using `$unset` on array elements expecting them to be removed - use `$pull` for that.
- Confusing `$unset` (removes field) with `$set: null` (keeps field with null value).

## Summary

`$unset` completely removes a field and its value from a MongoDB document. Use dot notation for nested fields. Remember that `$unset` on an array element nullifies the element rather than removing it - use `$pull` to remove array elements. Combine `$unset` and `$set` in a single update for efficient schema migrations.
