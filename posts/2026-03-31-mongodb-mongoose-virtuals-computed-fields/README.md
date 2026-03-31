# How to Use Mongoose Virtuals for Computed Fields

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Virtual, Computed Field, Node.js

Description: Learn how to define Mongoose virtual properties to add computed fields to your documents without storing extra data in MongoDB.

---

## What Are Mongoose Virtuals?

Virtuals are properties that are computed from other fields but never stored in MongoDB. They exist only in your Node.js objects. Use them for:

- Combining `firstName` and `lastName` into `fullName`
- Computing `age` from a `birthDate` field
- Building URLs or formatted strings from stored values
- Formatting currency or units for display

## Defining a Basic Virtual

```javascript
const userSchema = new mongoose.Schema({
  firstName: { type: String, required: true },
  lastName:  { type: String, required: true },
  birthDate: Date
});

// Getter: computes fullName on access
userSchema.virtual('fullName').get(function () {
  return `${this.firstName} ${this.lastName}`;
});

// Getter: computes age from birthDate
userSchema.virtual('age').get(function () {
  if (!this.birthDate) return null;
  const diff = Date.now() - this.birthDate.getTime();
  return Math.floor(diff / (1000 * 60 * 60 * 24 * 365.25));
});

const User = mongoose.model('User', userSchema);

const user = new User({ firstName: 'Alice', lastName: 'Smith', birthDate: new Date('1990-06-15') });
console.log(user.fullName); // "Alice Smith"
console.log(user.age);      // computed from birthDate
```

## Virtual Setters

A setter parses a combined value back into the underlying fields:

```javascript
userSchema.virtual('fullName')
  .get(function () {
    return `${this.firstName} ${this.lastName}`;
  })
  .set(function (v) {
    const parts = v.split(' ');
    this.firstName = parts[0];
    this.lastName = parts.slice(1).join(' ');
  });

const user = new User({});
user.fullName = 'Bob Johnson';
console.log(user.firstName); // "Bob"
console.log(user.lastName);  // "Johnson"
```

## Including Virtuals in JSON Output

Virtuals are excluded from `toJSON()` and `toObject()` by default. Enable them:

```javascript
const userSchema = new mongoose.Schema(
  { firstName: String, lastName: String },
  {
    toJSON:   { virtuals: true },
    toObject: { virtuals: true }
  }
);
```

Now `user.toJSON()` and `res.json(user)` include virtual fields.

## Populate Virtuals (Virtual Joins)

Virtuals can perform a Mongoose populate - a foreign key join without embedding:

```javascript
const authorSchema = new mongoose.Schema({ name: String });

authorSchema.virtual('posts', {
  ref:          'Post',          // model to join
  localField:   '_id',           // field on Author
  foreignField: 'authorId',      // field on Post
  justOne:      false            // return array
});

const Author = mongoose.model('Author', authorSchema);

// Populate posts when querying an author
const author = await Author.findById(id).populate('posts');
console.log(author.posts); // array of Post documents
```

Populate virtuals are especially useful for `one-to-many` relationships where the foreign key lives on the child document.

## Virtual for Building URLs

```javascript
const imageSchema = new mongoose.Schema({
  filename: String,
  bucket:   { type: String, default: 'uploads' }
});

imageSchema.virtual('url').get(function () {
  return `https://cdn.example.com/${this.bucket}/${this.filename}`;
});
```

Compute URLs dynamically rather than storing them - when the CDN domain changes, update one line.

## Summary

Mongoose virtuals define computed properties on document instances that are never stored in MongoDB. Use `.get()` for read-only computed values, `.set()` to parse combined inputs back into fields, and populate virtuals for cross-collection joins. Enable `toJSON: { virtuals: true }` to include them in API responses. Virtuals keep your stored documents lean while exposing rich, derived representations to application code.
