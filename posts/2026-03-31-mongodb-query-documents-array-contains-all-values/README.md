# How to Query Documents Where an Array Contains All Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Query

Description: Learn how to use MongoDB's $all operator to find documents where an array field contains every value in a specified list.

---

When you need to find documents where an array field contains every element from a given set - not just one of them - MongoDB's `$all` operator provides a direct solution.

## The $all Operator

`$all` matches documents where the array contains all of the specified values, regardless of order or other elements present:

```javascript
// Find products tagged with BOTH "electronics" AND "sale"
db.products.find({
  tags: { $all: ["electronics", "sale"] }
});
```

A document with `tags: ["electronics", "featured", "sale", "clearance"]` matches because both `"electronics"` and `"sale"` are present. A document with `tags: ["electronics", "featured"]` does not match.

## Order Does Not Matter

`$all` is order-independent. Both of these produce identical results:

```javascript
db.products.find({ tags: { $all: ["sale", "electronics"] } });
db.products.find({ tags: { $all: ["electronics", "sale"] } });
```

## $all vs. $and for Array Membership

Using `$and` with individual equality checks achieves the same result:

```javascript
// These two queries are equivalent
db.products.find({ tags: { $all: ["electronics", "sale"] } });

db.products.find({
  $and: [
    { tags: "electronics" },
    { tags: "sale" }
  ]
});
```

`$all` is more concise and expressive. Under the hood, MongoDB may apply the same optimization, but `$all` communicates intent more clearly.

## Using $all with Subdocument Arrays

When array elements are embedded documents, combine `$all` with `$elemMatch` to require multiple conditions on individual elements:

```javascript
// Find orders that contain line items for both productId "P-001" and "P-002"
db.orders.find({
  lineItems: {
    $all: [
      { $elemMatch: { productId: "P-001" } },
      { $elemMatch: { productId: "P-002" } }
    ]
  }
});
```

Each `$elemMatch` inside `$all` must match at least one element in the array (they can be different elements).

## Checking Exact Array Contents

`$all` checks for inclusion, not equality. To find documents where an array contains exactly a given set of values, combine `$all` with a size check:

```javascript
// Find products where tags is exactly ["electronics", "sale"] (any order)
db.products.find({
  tags: {
    $all: ["electronics", "sale"],
    $size: 2
  }
});
```

## Practical Example: Role-Based Access

A common use case is permission or role checking - find users who have all required roles:

```javascript
// Find users who have BOTH "editor" and "reviewer" roles
db.users.find({
  roles: { $all: ["editor", "reviewer"] }
});
```

## Index Behavior with $all

A multikey index on the array field can accelerate `$all` queries. MongoDB uses the index to satisfy the first condition in `$all` and then filters for additional values:

```javascript
db.products.createIndex({ tags: 1 });
```

With this index, MongoDB finds documents matching the first tag via index scan, then applies remaining `$all` conditions as filters. The more selective the first condition, the faster the query.

## Summary

Use `$all` when you need documents whose array contains every element in a list. It is order-independent, supports subdocument arrays via nested `$elemMatch`, and works with multikey indexes. For exact array matching, combine `$all` with `$size` to also constrain the total number of elements.
