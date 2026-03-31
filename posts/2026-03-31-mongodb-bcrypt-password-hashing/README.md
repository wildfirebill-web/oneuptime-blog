# How to Use bcrypt for Password Hashing with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Bcrypt, Password, Security, Authentication

Description: Learn how to securely hash and verify passwords using bcrypt before storing them in MongoDB, including salting, cost factors, and Mongoose integration.

---

## Overview

Storing plain-text passwords in MongoDB is a critical security vulnerability. `bcrypt` is the industry-standard algorithm for password hashing. It incorporates a salt automatically and uses a configurable cost factor (work factor) that makes brute-force attacks computationally expensive.

## Installation

```bash
npm install bcrypt mongoose
```

For environments with build issues, use the pure-JavaScript alternative:

```bash
npm install bcryptjs
```

## Understanding bcrypt Parameters

The cost factor (rounds) determines how many times the hashing algorithm iterates. Each increment doubles the computation time:

| Rounds | Approximate hash time |
|--------|----------------------|
| 10     | ~100ms               |
| 12     | ~400ms               |
| 14     | ~1.5s                |

A value of 12 is recommended for most production applications.

## Basic Hashing and Verification

```javascript
const bcrypt = require('bcrypt');

const SALT_ROUNDS = 12;

// Hash a password
async function hashPassword(plaintext) {
  return bcrypt.hash(plaintext, SALT_ROUNDS);
}

// Verify a password
async function verifyPassword(plaintext, hash) {
  return bcrypt.compare(plaintext, hash);
}

// Usage
const hash = await hashPassword('mySecretPassword');
console.log(hash);
// $2b$12$...

const isValid = await verifyPassword('mySecretPassword', hash);
console.log(isValid); // true
```

## Mongoose Integration with Pre-Save Hook

The most common pattern is to hash the password automatically before saving a user document:

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt   = require('bcrypt');

const SALT_ROUNDS = 12;

const userSchema = new mongoose.Schema({
  email:    { type: String, required: true, unique: true, lowercase: true },
  password: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
});

// Hash password before saving
userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, SALT_ROUNDS);
  next();
});

// Add a method to verify passwords
userSchema.methods.verifyPassword = function (plaintext) {
  return bcrypt.compare(plaintext, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

## Registration and Login Flow

```javascript
const User = require('./models/User');

// Register
async function register(email, password) {
  const user = await User.create({ email, password });
  // Password is hashed automatically by the pre-save hook
  return user;
}

// Login
async function login(email, password) {
  const user = await User.findOne({ email });
  if (!user) throw new Error('Invalid credentials');

  const isValid = await user.verifyPassword(password);
  if (!isValid) throw new Error('Invalid credentials');

  return user;
}
```

## Never Expose the Hash

When returning user objects in API responses, omit the password hash:

```javascript
userSchema.methods.toJSON = function () {
  const obj = this.toObject();
  delete obj.password;
  return obj;
};
```

Or use a projection in queries:

```javascript
const user = await User.findOne({ email }).select('-password');
```

## Handling Password Updates

When a user changes their password, the pre-save hook handles rehashing automatically because `isModified('password')` will be true:

```javascript
const user = await User.findById(userId);
user.password = newPassword;
await user.save(); // Triggers pre-save hook
```

## Summary

bcrypt is the safest choice for password hashing in MongoDB applications. Use a cost factor of 12 or higher, leverage Mongoose's pre-save hook to hash passwords automatically, and always use `bcrypt.compare()` for verification - never compare hashes manually. Exclude the password field from all API responses to prevent accidental exposure.
