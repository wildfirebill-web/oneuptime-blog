# How to Use $bit to Perform Bitwise AND, OR, and XOR Updates in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Update

Description: Learn how MongoDB's $bit update operator performs in-place bitwise AND, OR, and XOR operations on integer fields for flag and permission management.

---

MongoDB's `$bit` update operator applies a bitwise operation to an integer field directly in the database. This is particularly useful for managing feature flags, permissions bitmasks, and status flags without a read-modify-write cycle in your application.

## Syntax

```javascript
db.collection.updateOne(
  { filter },
  { $bit: { fieldName: { <operation>: <value> } } }
)
```

`<operation>` must be one of: `and`, `or`, `xor`.
The field and the value must both be integers.

## Bitwise AND - Clearing Bits (Removing Flags)

AND with a mask to turn off specific bits. For example, clear permission bit 2 (value 4) from a permissions field:

```javascript
// permissions = 0b0111 (7) - has bits 0, 1, 2 set
db.users.updateOne(
  { _id: "user_1" },
  { $bit: { permissions: { and: 0b11111011 } } }  // mask clears bit 2
)
// Result: permissions = 0b0011 (3)
```

In decimal:

```javascript
db.users.updateOne(
  { _id: "user_1" },
  { $bit: { permissions: { and: 251 } } }
)
```

## Bitwise OR - Setting Bits (Adding Flags)

OR with a value to turn on specific bits:

```javascript
// permissions = 0b0001 (1) - only bit 0 set
db.users.updateOne(
  { _id: "user_2" },
  { $bit: { permissions: { or: 4 } } }  // set bit 2
)
// Result: permissions = 0b0101 (5)
```

## Bitwise XOR - Toggling Bits

XOR flips specific bits - if a bit is on, it turns off; if it is off, it turns on:

```javascript
// flags = 0b1010 (10)
db.features.updateOne(
  { featureId: "dark-mode" },
  { $bit: { flags: { xor: 2 } } }  // toggle bit 1
)
// Result: flags = 0b1000 (8)
```

XOR the same value again to restore the original:

```javascript
db.features.updateOne(
  { featureId: "dark-mode" },
  { $bit: { flags: { xor: 2 } } }
)
// Result: flags = 0b1010 (10) - restored
```

## Practical Example - Feature Flag Bitmask

Store user feature flags as a single integer field, with each bit representing one feature:

```javascript
const FLAGS = {
  BETA_UI:       1,   // bit 0
  DARK_MODE:     2,   // bit 1
  NOTIFICATIONS: 4,   // bit 2
  ANALYTICS:     8    // bit 3
}

// Enable DARK_MODE and NOTIFICATIONS for a user
db.users.updateOne(
  { _id: "user_10" },
  { $bit: { featureFlags: { or: FLAGS.DARK_MODE | FLAGS.NOTIFICATIONS } } }
)

// Disable BETA_UI
db.users.updateOne(
  { _id: "user_10" },
  { $bit: { featureFlags: { and: ~FLAGS.BETA_UI } } }
)
```

## Querying by Bit Flags

After storing bitmasks, query with `$bitsAllSet` to find users with specific flags enabled:

```javascript
db.users.find({ featureFlags: { $bitsAllSet: 6 } })  // bits 1 and 2 set
```

## Limitations

- `$bit` only works on integer fields (32-bit and 64-bit integers).
- It does not work on floating-point numbers.
- The operation is atomic, so no separate read is needed.

## Summary

`$bit` performs atomic bitwise AND, OR, and XOR operations on integer fields in MongoDB. It is ideal for managing permission bitmasks and feature flags with a single database write. Use AND to clear bits, OR to set bits, and XOR to toggle bits. Pair with `$bitsAllSet` and related query operators for efficient flag-based filtering.
