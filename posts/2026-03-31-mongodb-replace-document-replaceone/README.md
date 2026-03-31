# How to Replace a Document in MongoDB with replaceOne()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, replaceOne, Update, Document Replacement, Write Operation

Description: Learn how replaceOne() differs from updateOne() and when to use full document replacement in MongoDB with practical examples.

---

## What Is replaceOne()

`replaceOne()` replaces the entire content of the first matching document with a new document. Unlike `updateOne()` with `$set`, it does not merge fields - it overwrites the document completely, preserving only the `_id`.

Signature:

```javascript
db.collection.replaceOne(filter, replacement, options)
```

- `filter` - query to find the document to replace
- `replacement` - the new document (must not contain update operators)
- `options` - e.g., `upsert: true`

## Basic Example

```javascript
db.users.replaceOne(
  { email: "alice@example.com" },
  {
    email: "alice@example.com",
    name: "Alice Smith",
    role: "admin",
    createdAt: new Date("2024-01-15"),
    updatedAt: new Date()
  }
)
```

The matched document is completely replaced. Any fields not in the replacement document (such as old `preferences` or `address` fields) are removed.

## What Gets Preserved

Only `_id` is preserved from the original document. If your replacement document includes `_id`, it must match the original, otherwise MongoDB throws an error.

```javascript
// This is fine - _id matches original
db.users.replaceOne(
  { _id: userId },
  { _id: userId, name: "Bob", email: "bob@example.com" }
)

// This throws - cannot change _id
db.users.replaceOne(
  { _id: userId },
  { _id: newId, name: "Bob" }
)
```

## replaceOne() vs updateOne()

| Feature | replaceOne() | updateOne() |
|---------|-------------|-------------|
| Uses update operators | No | Yes |
| Removes unspecified fields | Yes | No |
| Preserves `_id` | Yes | Yes |
| Best for | Full document swap | Partial field updates |

Use `replaceOne()` when you want a clean slate. Use `updateOne()` when you only need to change specific fields.

## Upsert with replaceOne()

```javascript
const result = await db.collection("configs").replaceOne(
  { key: "featureFlags" },
  { key: "featureFlags", flags: { darkMode: true, beta: false }, updatedAt: new Date() },
  { upsert: true }
);

if (result.upsertedCount > 0) {
  console.log("Document created with id:", result.upsertedId);
}
```

## Checking the Result

```javascript
const result = await db.collection("profiles").replaceOne(
  { userId: "u42" },
  newProfileDoc
);

console.log(result.matchedCount);  // 1 if found
console.log(result.modifiedCount); // 1 if replaced (0 if doc was identical)
```

## Common Mistakes

- Including update operators (`$set`, `$inc`) in the replacement document - this causes an error.
- Forgetting to include required fields in the replacement, effectively deleting them.
- Using `replaceOne()` when `updateOne()` would be safer for partial changes.

## Summary

`replaceOne()` completely overwrites a matched document with a new one, preserving only `_id`. It is the right tool when you need a total document swap rather than a partial field update. Always include all required fields in the replacement document, since any fields omitted from the new document will be gone after the operation.
