# How to Use Mongoose Timestamps (createdAt, updatedAt)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Timestamp, Schema, Audit

Description: Learn how to enable and customize Mongoose automatic timestamps to track document creation and update times with minimal configuration.

---

## Overview

Mongoose provides a built-in `timestamps` schema option that automatically adds and manages `createdAt` and `updatedAt` fields on every document. Enabling timestamps saves you from manually setting these fields in save hooks.

## Enabling Timestamps

Pass `{ timestamps: true }` as the second argument to `Schema`:

```javascript
const { Schema, model } = require('mongoose');

const postSchema = new Schema(
  {
    title:   { type: String, required: true },
    content: String,
    author:  String
  },
  { timestamps: true } // adds createdAt and updatedAt automatically
);

const Post = model('Post', postSchema);
```

## What Gets Added

```json
{
  "_id": "...",
  "title": "Hello World",
  "createdAt": "2026-03-31T10:00:00.000Z",
  "updatedAt": "2026-03-31T10:05:00.000Z"
}
```

- `createdAt` is set once when the document is first inserted.
- `updatedAt` is updated on every `save`, `updateOne`, `updateMany`, `findOneAndUpdate`, and `bulkWrite`.

## Customizing Field Names

Rename the timestamp fields to match your naming convention:

```javascript
const taskSchema = new Schema(
  { title: String, done: Boolean },
  {
    timestamps: {
      createdAt: 'created_at',
      updatedAt: 'updated_at'
    }
  }
);
```

## Using Timestamps with Updates

Timestamps are applied automatically on update queries:

```javascript
await Post.updateOne(
  { _id: postId },
  { $set: { title: 'Updated Title' } }
);
// updatedAt is set automatically - no need to include it in $set
```

## Querying by Timestamp

Index timestamps for efficient time-range queries:

```javascript
const auditSchema = new Schema(
  { action: String, userId: Schema.Types.ObjectId },
  { timestamps: true }
);

// Add an index for efficient querying
auditSchema.index({ createdAt: 1 });

// Query documents created in a time range
const recent = await Audit.find({
  createdAt: {
    $gte: new Date('2026-01-01'),
    $lt:  new Date('2026-04-01')
  }
}).sort({ createdAt: -1 });
```

## Handling createdAt in findOneAndUpdate with Upsert

When using upsert with `findOneAndUpdate`, Mongoose sets `createdAt` only on insert:

```javascript
const result = await User.findOneAndUpdate(
  { email: 'alice@example.com' },
  { $set: { name: 'Alice' } },
  { upsert: true, new: true, setDefaultsOnInsert: true }
);
// createdAt is set if inserted, updatedAt is always updated
```

## Disabling updatedAt on Specific Saves

Pass `timestamps: false` to `save` options to skip updating `updatedAt`:

```javascript
await doc.save({ timestamps: false });
```

## Summary

The Mongoose `timestamps` schema option automatically manages `createdAt` and `updatedAt` fields across all create and update operations. Customize field names with `{ timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' } }`. Add indexes on timestamp fields for time-range query performance. Use `save({ timestamps: false })` when you need to update a document without changing `updatedAt`.
