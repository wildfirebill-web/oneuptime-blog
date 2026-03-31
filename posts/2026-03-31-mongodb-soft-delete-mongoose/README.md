# How to Implement Soft Delete with Mongoose in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Soft Delete, Schema, Pattern

Description: Learn how to implement soft delete in Mongoose so deleted documents are marked but not removed, enabling recovery and audit trails in your MongoDB app.

---

## Overview

Soft delete marks documents as deleted without physically removing them from the database. This pattern preserves data for auditing, recovery, and referential integrity. In Mongoose, it is implemented using a `deletedAt` field combined with query middleware to automatically exclude deleted documents.

## Schema Setup

Add a `deletedAt` field to your schema:

```javascript
const { Schema, model } = require('mongoose');

const userSchema = new Schema({
  name:      { type: String, required: true },
  email:     { type: String, required: true },
  deletedAt: { type: Date, default: null }
});
```

## Instance Method for Soft Delete

```javascript
userSchema.methods.softDelete = async function() {
  this.deletedAt = new Date();
  return this.save();
};

userSchema.methods.restore = async function() {
  this.deletedAt = null;
  return this.save();
};

userSchema.methods.isDeleted = function() {
  return this.deletedAt !== null;
};
```

## Static Method for Bulk Soft Delete

```javascript
userSchema.statics.softDeleteMany = function(filter) {
  return this.updateMany(filter, { $set: { deletedAt: new Date() } });
};
```

## Automatic Filtering with Pre-Find Middleware

Use a pre-find hook to automatically exclude soft-deleted documents:

```javascript
userSchema.pre(/^find/, function(next) {
  // Only apply filter if not explicitly including deleted
  if (!this.getOptions().withDeleted) {
    this.where({ deletedAt: null });
  }
  next();
});

userSchema.pre('countDocuments', function(next) {
  if (!this.getOptions().withDeleted) {
    this.where({ deletedAt: null });
  }
  next();
});
```

## Using the Model

```javascript
const User = model('User', userSchema);

// Soft delete a user
const user = await User.findById(userId);
await user.softDelete();

// Normal find - soft-deleted users are excluded automatically
const activeUsers = await User.find({});

// Include soft-deleted users explicitly
const allUsers = await User.find({}).setOptions({ withDeleted: true });

// Restore a soft-deleted user
const deletedUser = await User.findById(userId).setOptions({ withDeleted: true });
await deletedUser.restore();
```

## Adding an Index for Performance

Index `deletedAt` to speed up the automatic filter:

```javascript
userSchema.index({ deletedAt: 1 });

// Compound index for common queries
userSchema.index({ email: 1, deletedAt: 1 });
```

## Encapsulating as a Plugin

```javascript
function softDeletePlugin(schema) {
  schema.add({ deletedAt: { type: Date, default: null } });
  schema.index({ deletedAt: 1 });

  schema.methods.softDelete = async function() {
    this.deletedAt = new Date();
    return this.save();
  };

  schema.pre(/^find/, function(next) {
    if (!this.getOptions().withDeleted) {
      this.where({ deletedAt: null });
    }
    next();
  });
}

module.exports = softDeletePlugin;
```

## Summary

Soft delete in Mongoose uses a `deletedAt` field set to the deletion timestamp instead of removing the document. A pre-find middleware automatically filters out deleted records unless the `withDeleted` option is passed. Encapsulate the pattern in a Mongoose plugin to apply it consistently across multiple models. Always index `deletedAt` to avoid full collection scans.
