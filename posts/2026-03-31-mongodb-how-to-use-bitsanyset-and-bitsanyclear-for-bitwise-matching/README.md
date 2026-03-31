# How to Use $bitsAnySet and $bitsAnyClear for Bitwise Matching in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Bitwise Operators, Query Operators, Database, NoSQL

Description: Learn how to use MongoDB's $bitsAnySet and $bitsAnyClear operators to query documents where any of the specified bit positions are set or clear.

---

## Overview

MongoDB provides four bitwise query operators for working with numeric bitmask fields:

- `$bitsAllSet` - all specified bits are 1
- `$bitsAllClear` - all specified bits are 0
- `$bitsAnySet` - at least one specified bit is 1
- `$bitsAnyClear` - at least one specified bit is 0

This post focuses on `$bitsAnySet` and `$bitsAnyClear`, which use OR logic rather than AND logic.

## $bitsAnySet

Matches documents where at least one of the specified bit positions is set to `1`.

```javascript
{ field: { $bitsAnySet: <bitmask or array> } }
```

### Example

```javascript
// Bit positions: 0=read(1), 1=write(2), 2=delete(4), 3=admin(8)
{ _id: 1, name: "Alice", permissions: 12 }  // binary: 1100 - delete + admin
{ _id: 2, name: "Bob",   permissions: 3  }  // binary: 0011 - read + write
{ _id: 3, name: "Carol", permissions: 0  }  // binary: 0000 - no permissions
```

Find users with any elevated permission (delete or admin):

```javascript
// Check if bit 2 (delete=4) or bit 3 (admin=8) is set - bitmask = 12
db.users.find({ permissions: { $bitsAnySet: 12 } })
// Or using array notation:
db.users.find({ permissions: { $bitsAnySet: [2, 3] } })
```

Returns Alice (permissions: 12) since she has both bits 2 and 3 set.

## $bitsAnyClear

Matches documents where at least one of the specified bit positions is clear (set to `0`).

```javascript
{ field: { $bitsAnyClear: <bitmask or array> } }
```

### Example

Using the same collection, find users who are missing at least one basic permission (read or write):

```javascript
// Check if bit 0 (read=1) or bit 1 (write=2) is clear - bitmask = 3
db.users.find({ permissions: { $bitsAnyClear: 3 } })
// Or:
db.users.find({ permissions: { $bitsAnyClear: [0, 1] } })
```

Returns Alice (permissions: 12, missing read and write) and Carol (permissions: 0, missing everything).

## Practical Use Case - Status Flags in IoT

IoT devices often use byte-packed status fields:

```javascript
// Status byte bits:
// 0: online    1: charging   2: low_battery   3: error
{ deviceId: "d1", status: 5  }  // binary: 0101 - online + low_battery
{ deviceId: "d2", status: 3  }  // binary: 0011 - online + charging
{ deviceId: "d3", status: 9  }  // binary: 1001 - online + error
{ deviceId: "d4", status: 0  }  // binary: 0000 - offline
```

Find devices with any alert (low_battery bit 2 or error bit 3):

```javascript
db.devices.find({ status: { $bitsAnySet: [2, 3] } })
```

Returns devices d1 and d3.

Find devices missing any required state (online bit 0 or charging bit 1):

```javascript
db.devices.find({ status: { $bitsAnyClear: [0, 1] } })
```

Returns d1, d3, and d4.

## Combining Both Operators

```javascript
// Find devices that have an error AND are missing at least one good state
db.devices.find({
  $and: [
    { status: { $bitsAnySet: [3] } },    // has error
    { status: { $bitsAnyClear: [0, 1] } } // missing online or charging
  ]
})
```

## Using BinData

Both operators support BinData fields:

```javascript
db.messages.find({
  flags: { $bitsAnySet: BinData(0, "Dg==") }
})
```

## Supported Types

Both `$bitsAnySet` and `$bitsAnyClear` work with:
- Integer values (int32, int64)
- BinData

They do not apply to floating-point numbers.

## Summary

`$bitsAnySet` matches documents where at least one specified bit is 1, while `$bitsAnyClear` matches where at least one specified bit is 0. Together with `$bitsAllSet` and `$bitsAllClear`, these operators give you full bitwise querying capability in MongoDB, making them ideal for permission systems, status flags, and IoT telemetry data.
