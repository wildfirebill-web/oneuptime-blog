# How to Use Mongoose Virtual Properties

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Virtual, Schema, Node.js

Description: Learn how to define Mongoose virtual properties to create computed fields on documents without storing them in MongoDB.

---

## Overview

Virtual properties are document fields that Mongoose computes on the fly. They are not stored in MongoDB but behave like normal document properties. Virtuals are ideal for derived values like full names, formatted dates, or computed URLs.

## Defining a Basic Virtual

```javascript
const { Schema, model } = require('mongoose');

const userSchema = new Schema({
  firstName: String,
  lastName:  String,
  email:     String
});

// Getter virtual
userSchema.virtual('fullName').get(function() {
  return `${this.firstName} ${this.lastName}`;
});

const User = model('User', userSchema);
const user = new User({ firstName: 'Jane', lastName: 'Doe' });
console.log(user.fullName); // "Jane Doe"
```

## Getter and Setter Virtuals

A virtual can also have a setter to parse a combined value back to stored fields:

```javascript
userSchema.virtual('fullName')
  .get(function() {
    return `${this.firstName} ${this.lastName}`;
  })
  .set(function(name) {
    const parts = name.split(' ');
    this.firstName = parts[0];
    this.lastName = parts.slice(1).join(' ');
  });

const user = new User({});
user.fullName = 'John Smith';
console.log(user.firstName); // "John"
console.log(user.lastName);  // "Smith"
```

## Including Virtuals in JSON and Object Output

By default, virtuals are excluded from `toJSON()` and `toObject()`. Enable them in the schema options:

```javascript
const personSchema = new Schema(
  { firstName: String, lastName: String },
  { toJSON: { virtuals: true }, toObject: { virtuals: true } }
);

personSchema.virtual('fullName').get(function() {
  return `${this.firstName} ${this.lastName}`;
});
```

Now `JSON.stringify(person)` includes `fullName`.

## Virtual Populate (Joining References)

Virtual populate lets you populate related documents without storing the reference in the parent:

```javascript
const blogSchema = new Schema({ title: String, authorId: Schema.Types.ObjectId });

blogSchema.virtual('comments', {
  ref:          'Comment',
  localField:   '_id',
  foreignField: 'postId'
});

const Blog = model('Blog', blogSchema);

// Populate virtual
const blog = await Blog.findById(id).populate('comments');
console.log(blog.comments); // Array of Comment documents
```

## Computed URL Virtual

```javascript
const productSchema = new Schema({ slug: String });

productSchema.virtual('url').get(function() {
  return `https://example.com/products/${this.slug}`;
});
```

## When to Use Virtuals vs. Stored Fields

```text
Use virtuals for:
  - Computed values (full name, formatted price)
  - Reverse population without storing extra refs
  - API response shaping

Use stored fields for:
  - Values you need to query or index
  - Values that are expensive to recompute
```

## Summary

Mongoose virtuals define computed properties on documents without persisting data to MongoDB. Use getter-only virtuals for derived values, getter-setter pairs for two-way binding, and virtual populate for joining related documents by reference. Enable `{ toJSON: { virtuals: true } }` in schema options to include virtuals in serialized output.
