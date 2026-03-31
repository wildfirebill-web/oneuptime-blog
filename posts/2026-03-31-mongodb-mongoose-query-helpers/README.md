# How to Use Mongoose Query Helpers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Query, Helper, Schema

Description: Learn how to define Mongoose query helpers to create reusable, chainable query conditions that keep your query logic DRY and readable.

---

## Overview

Mongoose query helpers are chainable methods added to the query builder for a specific schema. Unlike static methods that return a promise, query helpers return the query object itself, enabling method chaining. They are the Mongoose equivalent of named scopes in ActiveRecord.

## Defining Query Helpers

Add query helpers to `schema.query`:

```javascript
const { Schema, model } = require('mongoose');

const userSchema = new Schema({
  name:      String,
  email:     String,
  role:      { type: String, enum: ['admin', 'user', 'guest'], default: 'user' },
  active:    { type: Boolean, default: true },
  createdAt: { type: Date, default: Date.now }
});

// Filter by active status
userSchema.query.active = function() {
  return this.where({ active: true });
};

// Filter by role
userSchema.query.byRole = function(role) {
  return this.where({ role });
};

// Sort by creation date (newest first)
userSchema.query.newest = function() {
  return this.sort({ createdAt: -1 });
};

// Limit and paginate
userSchema.query.page = function(pageNum, pageSize = 20) {
  return this.skip((pageNum - 1) * pageSize).limit(pageSize);
};

const User = model('User', userSchema);
```

## Chaining Query Helpers

Query helpers chain together and with standard Mongoose query methods:

```javascript
// Get active admins, sorted newest first
const admins = await User.find()
  .active()
  .byRole('admin')
  .newest()
  .lean();

// Paginated active users
const page2Users = await User.find()
  .active()
  .newest()
  .page(2, 10);
```

## Date Range Helper

```javascript
userSchema.query.createdBetween = function(start, end) {
  return this.where({ createdAt: { $gte: start, $lte: end } });
};

const q1Users = await User.find()
  .createdBetween(new Date('2026-01-01'), new Date('2026-03-31'))
  .active()
  .lean();
```

## Search Helper

```javascript
userSchema.query.search = function(term) {
  const regex = new RegExp(term, 'i');
  return this.where({ $or: [{ name: regex }, { email: regex }] });
};

const results = await User.find().search('alice').active().limit(5);
```

## Query Helpers vs Static Methods

```text
Use query helpers when:
  - The result needs further query chaining
  - Defining reusable filter conditions
  - Building composable query logic

Use static methods when:
  - Returning the final result directly
  - Performing operations beyond querying (aggregation, bulkWrite)
  - Returning non-query results
```

## Combining with Aggregation

Query helpers do not work with `aggregate()`. For aggregation, use static methods instead:

```javascript
userSchema.statics.activeAdminStats = function() {
  return this.aggregate([
    { $match: { active: true, role: 'admin' } },
    { $group: { _id: '$role', count: { $sum: 1 } } }
  ]);
};
```

## Summary

Mongoose query helpers are reusable filter and sort conditions attached to `schema.query`. They return the query object to allow chainability with other query helpers and standard Mongoose methods like `.limit()`, `.sort()`, and `.lean()`. Use them to define named conditions that keep query logic DRY and expressive throughout your application.
