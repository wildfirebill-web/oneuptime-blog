# How to Use Mongoose Instance Methods and Static Methods

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Method, Static, Schema

Description: Learn how to define Mongoose instance methods and static methods to encapsulate document-level and collection-level business logic in your models.

---

## Overview

Mongoose lets you attach custom methods to schema definitions. Instance methods operate on a single document (`this` is the document), while static methods operate on the model class (`this` is the model). Both approaches keep your business logic co-located with your data model.

## Instance Methods

Instance methods are functions available on every document instance:

```javascript
const bcrypt = require('bcryptjs');
const { Schema, model } = require('mongoose');

const userSchema = new Schema({
  name:     String,
  email:    String,
  password: String,
  role:     { type: String, default: 'user' }
});

// Compare password for login
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

// Return a public-safe profile object
userSchema.methods.toPublicProfile = function() {
  return {
    id:    this._id,
    name:  this.name,
    email: this.email,
    role:  this.role
  };
};

const User = model('User', userSchema);
```

Using instance methods:

```javascript
const user = await User.findOne({ email: 'alice@example.com' });
const isMatch = await user.comparePassword('secret123');
const profile = user.toPublicProfile();
```

## Static Methods

Static methods are available on the model class and operate over the collection:

```javascript
// Find by email (common lookup pattern)
userSchema.statics.findByEmail = async function(email) {
  return this.findOne({ email: email.toLowerCase() });
};

// Find all active admins
userSchema.statics.findAdmins = function() {
  return this.find({ role: 'admin', active: true });
};

// Count active users
userSchema.statics.countActive = function() {
  return this.countDocuments({ active: true });
};
```

Using static methods:

```javascript
const user   = await User.findByEmail('Alice@example.com');
const admins = await User.findAdmins();
const count  = await User.countActive();
```

## Using Arrow Functions vs Regular Functions

Always use regular `function` declarations for methods - arrow functions do not bind `this`:

```javascript
// WRONG: arrow function - 'this' is undefined
userSchema.methods.getName = () => this.name;

// CORRECT: regular function - 'this' is the document
userSchema.methods.getName = function() { return this.name; };
```

## Chaining Statics with Query Builder

Statics can return query builder objects for chaining:

```javascript
userSchema.statics.active = function() {
  return this.find({ active: true });
};

// Chain additional query methods
const recentActive = await User.active()
  .sort({ createdAt: -1 })
  .limit(10)
  .lean();
```

## TypeScript Typing for Methods

```typescript
interface IUserMethods {
  comparePassword(candidate: string): Promise<boolean>;
  toPublicProfile(): Record<string, unknown>;
}

interface IUserModel extends mongoose.Model<IUser, {}, IUserMethods> {
  findByEmail(email: string): Promise<IUser | null>;
}
```

## Summary

Mongoose instance methods are attached to `schema.methods` and operate on individual documents using `this` as the document instance. Static methods are attached to `schema.statics` and operate on the model class. Use regular function declarations (not arrow functions) to preserve `this` binding. Together, these features keep business logic encapsulated within your data model.
