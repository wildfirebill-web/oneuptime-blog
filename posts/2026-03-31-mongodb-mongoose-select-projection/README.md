# How to Use Mongoose Select and Projection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Projection, SELECT, Performance

Description: Learn how to use Mongoose select() and schema-level projection to return only the fields you need, reducing query overhead and protecting sensitive data.

---

## Why Projection Matters

Fetching entire documents when you only need two or three fields wastes bandwidth, increases deserialization time, and may accidentally expose sensitive data like passwords or tokens. Projection tells MongoDB which fields to include or exclude in the query result.

## Using .select() on a Query

Pass a space-separated string of field names to `select()`:

```javascript
// Include only username and email
const user = await User.findById(userId).select('username email');

// Exclude the password field (all other fields are returned)
const user2 = await User.findById(userId).select('-password -refreshToken');
```

Prefix a field with `-` to exclude it. You cannot mix inclusion and exclusion in a single `select()` call (except for `_id`).

## Selecting Nested Fields

Use dot notation to project nested fields:

```javascript
const order = await Order.findById(orderId)
  .select('status items.productId items.quantity -_id');
```

## Chaining select() with populate()

`select()` and `populate()` combine cleanly:

```javascript
const post = await Post
  .findById(postId)
  .select('title body createdAt')
  .populate('author', 'username avatar -_id');
```

The second argument to `populate()` is a field selection string for the populated document.

## Schema-Level Default Projection

Use `select: false` in the schema to exclude sensitive fields from all queries by default:

```javascript
const userSchema = new mongoose.Schema({
  username: String,
  email:    String,
  password: { type: String, select: false },         // excluded by default
  twoFactorSecret: { type: String, select: false }   // excluded by default
});
```

With `select: false`, password is never returned unless explicitly requested:

```javascript
// Normal query - no password
const user = await User.findById(id);
console.log(user.password); // undefined

// Explicitly include it when needed (e.g., for authentication)
const userWithPw = await User.findOne({ email }).select('+password');
```

The `+` prefix overrides `select: false` for that specific query.

## Using Object Syntax for Projection

Pass a plain object to `select()` for field-level granularity:

```javascript
const result = await Product.find({ inStock: true })
  .select({ name: 1, price: 1, _id: 0 });
```

`1` includes, `0` excludes, `_id: 0` explicitly removes the default `_id` inclusion.

## Projection with .lean()

`lean()` returns plain JavaScript objects and is faster than Mongoose documents. Combine with `select` for maximum efficiency in read-only routes:

```javascript
const products = await Product.find({ category: 'electronics' })
  .select('name price stock')
  .lean()
  .limit(50);
```

`lean()` skips Mongoose document hydration - results cannot use Mongoose methods but serialize faster.

## Projection in Aggregation Pipelines

Use `$project` inside aggregation for more complex field transformations:

```javascript
const results = await Order.aggregate([
  { $match: { status: 'completed' } },
  { $project: {
    orderId: '$_id',
    total: 1,
    itemCount: { $size: '$items' },
    _id: 0
  }}
]);
```

## Summary

Use `.select('field1 field2')` to include specific fields or `.select('-field')` to exclude them. Mark sensitive fields with `select: false` in the schema to protect them by default, and override with `+fieldName` when genuinely needed. Combine `select` with `lean()` for high-performance read routes. Use `$project` in aggregation pipelines for computed projections. Consistent projection reduces payload size, improves latency, and prevents accidental data leakage.
