# How to Use $bitsAnySet and $bitsAnyClear for Bitwise Matching in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $bitsAnySet, $bitsAnyClear, Bitwise Query, Query Operator

Description: Learn how to use MongoDB's $bitsAnySet and $bitsAnyClear operators to find documents where at least one specified bit is set or cleared in a numeric field.

---

## What Are $bitsAnySet and $bitsAnyClear

MongoDB provides four bitwise query operators. The "Any" variants match when at least one of the specified bits matches the condition:

- `$bitsAnySet` - matches if ANY specified bit is `1` (set)
- `$bitsAnyClear` - matches if ANY specified bit is `0` (clear)

These complement `$bitsAllSet` and `$bitsAllClear`, which require ALL bits to match.

Syntax:

```javascript
{ field: { $bitsAnySet: bitmask } }
{ field: { $bitsAnyClear: bitmask } }
```

## Understanding the Difference

```javascript
const PERMISSIONS = {
  READ: 1,    // bit 0
  WRITE: 2,   // bit 1
  DELETE: 4,  // bit 2
  ADMIN: 8    // bit 3
};

const mask = PERMISSIONS.WRITE | PERMISSIONS.DELETE; // 6

// $bitsAllSet: must have BOTH write AND delete
db.users.find({ permissions: { $bitsAllSet: mask } })

// $bitsAnySet: must have write OR delete (or both)
db.users.find({ permissions: { $bitsAnySet: mask } })

// $bitsAllClear: must have NEITHER write NOR delete
db.users.find({ permissions: { $bitsAllClear: mask } })

// $bitsAnyClear: write OR delete (or both) are missing
db.users.find({ permissions: { $bitsAnyClear: mask } })
```

## $bitsAnySet Examples

Find users who have at least one elevated permission:

```javascript
const elevatedMask = PERMISSIONS.WRITE | PERMISSIONS.DELETE | PERMISSIONS.ADMIN;

db.users.find({
  permissions: { $bitsAnySet: elevatedMask }
})
```

Find devices with at least one warning flag:

```javascript
const WARNING_FLAGS = {
  LOW_BATTERY: 1,
  HIGH_TEMP: 2,
  CONNECTIVITY_ISSUE: 4,
  STORAGE_LOW: 8
};

const anyWarning = WARNING_FLAGS.LOW_BATTERY | WARNING_FLAGS.HIGH_TEMP |
                   WARNING_FLAGS.CONNECTIVITY_ISSUE | WARNING_FLAGS.STORAGE_LOW;

db.devices.find({ alertFlags: { $bitsAnySet: anyWarning } })
```

## $bitsAnyClear Examples

Find users missing at least one required permission:

```javascript
const requiredPermissions = PERMISSIONS.READ | PERMISSIONS.WRITE;

// Users who are missing READ or WRITE (or both)
db.users.find({
  permissions: { $bitsAnyClear: requiredPermissions }
})
```

Find devices that are missing at least one required feature:

```javascript
const requiredFeatures = 0b00001111; // bits 0-3 required

db.devices.find({
  capabilities: { $bitsAnyClear: requiredFeatures }
})
```

## Using Bit Position Arrays

Specify bit positions as an array instead of a bitmask:

```javascript
// Check positions 1 and 2 (write and delete)
db.users.find({ permissions: { $bitsAnySet: [1, 2] } })
db.users.find({ permissions: { $bitsAnyClear: [1, 2] } })
```

## Combining Bitwise Operators

```javascript
// Users who have admin, but are missing either read or write
db.users.find({
  $and: [
    { permissions: { $bitsAllSet: PERMISSIONS.ADMIN } },
    { permissions: { $bitsAnyClear: PERMISSIONS.READ | PERMISSIONS.WRITE } }
  ]
})
```

## All Four Bitwise Operators Summary

| Operator | Logic | Description |
|----------|-------|-------------|
| `$bitsAllSet` | ALL bits are 1 | Every specified bit is set |
| `$bitsAllClear` | ALL bits are 0 | Every specified bit is clear |
| `$bitsAnySet` | ANY bit is 1 | At least one specified bit is set |
| `$bitsAnyClear` | ANY bit is 0 | At least one specified bit is clear |

## Performance Considerations

All bitwise operators require evaluating the expression per document. They do not use standard B-tree indexes efficiently. For high-frequency queries:

1. Store individual boolean flags as separate indexed fields.
2. Pre-compute boolean results and store them as indexed fields.

```javascript
// Faster alternative for common queries
await db.collection("users").createIndex({ hasElevatedPerms: 1 });

// Pre-compute on write
await collection.updateOne(
  { _id: userId },
  { $set: { hasElevatedPerms: (newPermissions & elevatedMask) !== 0 } }
);
```

## Common Mistakes

- Confusing "Any" with "All" variants - "Any" needs only one bit to match.
- Using bitwise operators on non-integer types.
- Expecting index acceleration on bitwise queries without a computed field.

## Summary

`$bitsAnySet` matches documents where at least one specified bit is set, while `$bitsAnyClear` matches when at least one specified bit is clear. Together with `$bitsAllSet` and `$bitsAllClear`, they provide flexible bitwise filtering. For production performance on large collections, pre-compute and index boolean results rather than relying on bitwise queries at query time.
