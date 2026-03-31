# How to Handle Conflict Resolution in Atlas Device Sync

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Conflict Resolution, Mobile

Description: Learn how MongoDB Atlas Device Sync automatically resolves write conflicts using operational transformation and how to design your schema to minimize conflicts.

---

## Overview

When multiple devices write to the same synced Realm object while offline, conflicts arise when they reconnect. Atlas Device Sync resolves these conflicts automatically using a last-write-wins (LWW) strategy at the field level, based on server timestamps. Understanding this model helps you design schemas that minimize unwanted data loss.

## How Atlas Device Sync Resolves Conflicts

Device Sync uses **operational transformation (OT)** at the property level - not the document level. This means:

- If Device A updates `task.name` and Device B updates `task.isComplete` on the same object while offline, both changes are preserved.
- If both Device A and Device B update `task.name`, the update with the later server timestamp wins.

This is different from MongoDB's write conflict in transactions, where the later write retries. In Device Sync, conflicting property updates are resolved server-side without error.

## Last-Write-Wins at the Field Level

```text
Device A (offline): task.name = "Buy milk"  -> synced at T=100
Device B (offline): task.name = "Buy oat milk" -> synced at T=102

Result: task.name = "Buy oat milk"  (T=102 wins)
```

## Designing Schemas to Avoid Conflicts

### Use Additive Operations Instead of Overwrites

Instead of storing a counter as a plain integer that devices overwrite, use an append-only list:

```javascript
const EventSchema = {
  name: 'Event',
  primaryKey: '_id',
  properties: {
    _id: 'objectId',
    votes: { type: 'list', objectType: 'int' },
  },
};

// Each device appends a vote instead of incrementing a counter
realm.write(() => {
  event.votes.push(1);
});
```

The total count is then computed at read time: `event.votes.reduce((a, b) => a + b, 0)`.

### Use Separate Fields for Concurrent Updates

Instead of one `status` string updated by multiple clients, use per-device status fields:

```javascript
const OrderSchema = {
  name: 'Order',
  primaryKey: '_id',
  properties: {
    _id: 'objectId',
    driverStatus: 'string',
    warehouseStatus: 'string',
  },
};
```

Each role only updates its own field, eliminating conflicts entirely.

## Handling Compensating Writes

When Device Sync cannot apply a write due to a permission rule violation or schema mismatch, it issues a **compensating write** - reverting the local change on the client. Detect these in your error handler:

```javascript
// React Native example
<RealmProvider
  sync={{
    flexible: true,
    onError: (session, error) => {
      if (error.name === 'ClientResetError') {
        console.error('Client reset required:', error.message);
        // Trigger client reset flow
      }
    },
  }}
>
```

## Client Reset

In rare cases (such as a server-side rollback), Atlas Device Sync requires a **client reset**. Configure an automatic client reset strategy:

```javascript
const config = {
  schema: [TaskSchema],
  sync: {
    user,
    flexible: true,
    clientReset: {
      mode: 'recoverOrDiscardUnsyncedChanges',
    },
  },
};
```

The `recoverOrDiscardUnsyncedChanges` mode attempts to replay local changes after reset. If recovery fails, unsynced changes are discarded.

## Best Practices

- Prefer additive data patterns (append-only lists, separate per-actor fields) over shared mutable fields.
- Use small, focused object models so that field-level conflict resolution has less surface area.
- Monitor the Atlas App Services logs for compensating write events to identify client-server permission mismatches.

## Summary

Atlas Device Sync resolves write conflicts automatically at the property level using last-write-wins semantics. By designing schemas with additive operations and per-actor fields, you minimize the impact of concurrent writes. For edge cases, configure client reset strategies to recover gracefully.
