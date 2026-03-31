# How to Use Mongoose Validators (Built-In and Custom)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Validation, Schema, Node.js

Description: Learn how to use Mongoose built-in validators and write custom validator functions to enforce data integrity before documents are saved to MongoDB.

---

## Why Validate with Mongoose?

Mongoose validators run in JavaScript before any data reaches MongoDB. They catch errors early, provide meaningful error messages to clients, and reduce the need for application-level null checks.

MongoDB's own JSON Schema validation is the last line of defense; Mongoose validation is the first.

## Built-In Validators

Mongoose provides validators for each data type:

```javascript
const productSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Product name is required'],
    minlength: [2, 'Name must be at least 2 characters'],
    maxlength: [100, 'Name cannot exceed 100 characters'],
    trim: true
  },
  price: {
    type: Number,
    required: true,
    min: [0, 'Price cannot be negative'],
    max: [999999, 'Price exceeds maximum allowed']
  },
  sku: {
    type: String,
    match: [/^[A-Z]{2}-\d{4}$/, 'SKU must be in format XX-0000']
  },
  category: {
    type: String,
    enum: {
      values: ['electronics', 'clothing', 'food', 'books'],
      message: '{VALUE} is not a valid category'
    }
  },
  inStock: {
    type: Boolean,
    default: true
  }
});
```

The second element of an array validator is the custom error message, with `{VALUE}` as a placeholder for the actual value.

## Custom Validator Functions

Use the `validate` property for logic not covered by built-in validators:

```javascript
const userSchema = new mongoose.Schema({
  username: {
    type: String,
    validate: {
      validator: function(v) {
        return /^[a-z0-9_]{3,20}$/.test(v);
      },
      message: props => `${props.value} is not a valid username. Use 3-20 lowercase alphanumeric characters or underscores.`
    }
  },
  email: {
    type: String,
    validate: {
      validator: async function(v) {
        // Async validator: check uniqueness beyond the unique index
        const count = await mongoose.model('User').countDocuments({ email: v });
        return count === 0;
      },
      message: 'Email address is already in use'
    }
  }
});
```

Async validators return a Promise or use the `callback` parameter. They are powerful but add a database round-trip on save.

## Validating Array Elements

Apply validators to items within an array:

```javascript
const orderSchema = new mongoose.Schema({
  items: [{
    productId: { type: mongoose.Schema.Types.ObjectId, required: true },
    quantity: {
      type: Number,
      required: true,
      min: [1, 'Quantity must be at least 1'],
      validate: {
        validator: Number.isInteger,
        message: 'Quantity must be an integer'
      }
    }
  }]
});
```

## Triggering Validation Manually

Validation runs automatically on `save()` and `create()`. Trigger it manually without saving:

```javascript
const product = new Product({ name: 'x', price: -5 });

try {
  await product.validate();
} catch (err) {
  // err.errors is an object keyed by field path
  Object.entries(err.errors).forEach(([field, error]) => {
    console.log(`${field}: ${error.message}`);
  });
}
```

## Handling Validation Errors

```javascript
try {
  await product.save();
} catch (err) {
  if (err.name === 'ValidationError') {
    const messages = Object.values(err.errors).map(e => e.message);
    res.status(400).json({ errors: messages });
  } else {
    throw err;
  }
}
```

## Running Validators on Update

By default, validators do not run on `updateOne`, `findOneAndUpdate`, etc. Enable them:

```javascript
await Product.findOneAndUpdate(
  { _id: productId },
  { price: -10 },
  { runValidators: true, new: true }
);
// Throws ValidationError because price < 0
```

## Summary

Mongoose built-in validators (`required`, `min`, `max`, `minlength`, `maxlength`, `match`, `enum`) cover most common cases. Use the `validate` property for custom synchronous or async logic. Enable `runValidators: true` on update operations to enforce validation outside of `save()`. Handle `ValidationError` to return structured error responses. Mongoose validation at the ODM layer provides fast, descriptive errors before any database round-trip occurs.
