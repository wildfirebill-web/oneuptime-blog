# How to Remove a Field from All Documents in a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Schema, Migration, Operator

Description: Learn how to remove a field from all documents in a MongoDB collection using updateMany with $unset, including nested fields and safe migration patterns.

---

Removing obsolete or renamed fields from all documents in a collection is a routine MongoDB schema maintenance task. The `$unset` operator combined with `updateMany` efficiently removes a field from every matching document.

## Basic Field Removal

```javascript
// Remove the "legacyId" field from all documents
db.users.updateMany(
  {},
  { $unset: { legacyId: "" } }
)
```

The value passed to `$unset` is ignored - conventionally an empty string is used. The empty filter `{}` targets all documents.

## Removing a Field Only Where It Exists

Running `$unset` on a field that doesn't exist is a no-op - MongoDB won't throw an error. However, adding an existence check improves clarity and performance on large collections:

```javascript
db.users.updateMany(
  { legacyId: { $exists: true } },
  { $unset: { legacyId: "" } }
)
```

## Removing Multiple Fields at Once

```javascript
db.products.updateMany(
  {},
  {
    $unset: {
      deprecatedField: "",
      tempFlag: "",
      oldCategory: ""
    }
  }
)
```

## Removing a Nested Field

Use dot notation with `$unset` to remove a field within an embedded document:

```javascript
// Remove address.fax from all documents
db.customers.updateMany(
  {},
  { $unset: { "address.fax": "" } }
)
```

## Removing an Array Element by Position

`$unset` on an array index sets the element to `null` rather than removing it:

```javascript
// Sets items[1] to null (does NOT shrink the array)
db.orders.updateOne(
  { _id: orderId },
  { $unset: { "items.1": "" } }
)
```

To actually remove the null value from the array, follow with `$pull`:

```javascript
db.orders.updateOne(
  { _id: orderId },
  { $pull: { items: null } }
)
```

## Dropping a Field vs Setting to Null

Removing a field with `$unset` is different from setting it to `null`:

```javascript
// Field no longer exists in the document
db.users.updateOne({ _id: id }, { $unset: { phone: "" } })

// Field exists but is null - still occupies space, affects $exists queries
db.users.updateOne({ _id: id }, { $set: { phone: null } })
```

If you want the field absent from `$exists: false` queries, use `$unset`.

## Verifying the Migration

```javascript
// Confirm no documents still have the field
const remaining = await db.collection("users").countDocuments({
  legacyId: { $exists: true }
});

if (remaining === 0) {
  console.log("Migration complete");
} else {
  console.log(`${remaining} documents still have legacyId`);
}
```

## Dropping a Top-Level Index After Removing a Field

After removing a field from all documents, drop any indexes that referenced it:

```javascript
db.users.dropIndex("legacyId_1")
```

## Summary

Use `db.collection.updateMany({}, { $unset: { fieldName: "" } })` to remove a field from all documents. For nested fields, use dot notation. The `$unset` operation is safe to run multiple times - it is a no-op for documents that do not have the field. After removing the field from all documents, drop any indexes that referenced the removed field to reclaim storage.
