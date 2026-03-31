# How to Use $bitsAllClear for Bitwise Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $bitsAllClear, Bitwise Query, Query Operator, Bit Mask

Description: Learn how to use MongoDB's $bitsAllClear operator to find documents where all specified bits in a numeric field are unset (zero).

---

## What Is $bitsAllClear

The `$bitsAllClear` operator matches documents where all of the specified bit positions in a numeric or BinData field are clear (equal to 0). It is part of MongoDB's suite of bitwise query operators, useful for flag fields, permission masks, and feature toggles stored as integers.

Syntax (using a bitmask):

```javascript
{ field: { $bitsAllClear: bitmask } }
```

The `bitmask` specifies which bits to check. Bits at positions set to `1` in the bitmask must be `0` in the document's field.

## Understanding Bit Positions

```
Bit position:  7  6  5  4  3  2  1  0
Binary value: 128 64 32 16  8  4  2  1
```

A bitmask of `6` (binary `00000110`) checks bits at positions 1 and 2.

## Basic Example

```javascript
// Find documents where bits 1 and 2 are both clear (permission bits not set)
db.users.find({ permissions: { $bitsAllClear: 6 } })
// Matches documents where permissions & 6 === 0
```

## Storing Permissions as Bit Flags

```javascript
const PERMISSIONS = {
  READ: 1,       // bit 0: 00000001
  WRITE: 2,      // bit 1: 00000010
  DELETE: 4,     // bit 2: 00000100
  ADMIN: 8       // bit 3: 00001000
};

// Find users who have neither WRITE nor DELETE permissions
const writeAndDelete = PERMISSIONS.WRITE | PERMISSIONS.DELETE; // 6

db.users.find({
  permissions: { $bitsAllClear: writeAndDelete }
})
```

## Using an Array of Bit Positions

Instead of a bitmask integer, you can specify an array of bit position numbers:

```javascript
// Check that bits at positions 1 and 2 are both clear
db.users.find({ permissions: { $bitsAllClear: [1, 2] } })
```

Position 0 is the least significant bit.

## Using a BinData Bitmask

For large flag fields stored as BinData:

```javascript
db.flags.find({
  featureFlags: {
    $bitsAllClear: BinData(0, "AAAA")  // all bits in the mask must be 0
  }
})
```

## Practical Example: Feature Toggle System

```javascript
const FEATURES = {
  DARK_MODE: 1,
  BETA_ACCESS: 2,
  EXPERIMENTAL_UI: 4,
  DEBUG_MODE: 8
};

// Find users with no experimental features enabled
const experimentalMask = FEATURES.BETA_ACCESS | FEATURES.EXPERIMENTAL_UI | FEATURES.DEBUG_MODE;

const regularUsers = await db.collection("users").find({
  featureFlags: { $bitsAllClear: experimentalMask }
}).toArray();
```

## Combining $bitsAllClear with Other Operators

```javascript
// Active users who have no write or delete permissions
db.users.find({
  status: "active",
  permissions: { $bitsAllClear: PERMISSIONS.WRITE | PERMISSIONS.DELETE }
})
```

## $bitsAllClear vs $bitsAllSet

| Operator | Matches when |
|----------|-------------|
| `$bitsAllClear` | All specified bits are 0 |
| `$bitsAllSet` | All specified bits are 1 |

## Index Support

Bitwise query operators do not use standard indexes efficiently. They require evaluating the bit expression on each document. For performance-critical queries, consider storing boolean fields separately instead of packing flags into a single integer.

## Common Mistakes

- Confusing bitmask values with bit positions - `$bitsAllClear: 6` checks positions 1 and 2, not position 6.
- Expecting `$bitsAllClear: 0` to match everything - a bitmask of 0 means no bits are checked, so all documents match.
- Using bitwise queries on floating-point values - they only work reliably on integer types.

## Summary

`$bitsAllClear` matches documents where all bits specified by the bitmask are unset (0) in the field. Use it for permission systems and feature flag checks stored as integer bit fields. Specify the bitmask as an integer (OR of individual flag values) or as an array of bit positions. Keep in mind that bitwise queries do not benefit from standard indexes.
