# How to Perform CRUD Operations with the MongoDB Ruby Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Ruby, CRUD, Collection, Document

Description: A practical guide to creating, reading, updating, and deleting documents in MongoDB using the official Ruby driver with real code examples.

---

## Introduction

Once you have a `Mongo::Client` connected, performing CRUD operations is straightforward. The Ruby driver exposes a collection object whose methods map closely to the MongoDB wire protocol, giving you full control over how documents are manipulated.

## Setup

```ruby
require 'mongo'

client = Mongo::Client.new(['localhost:27017'], database: 'shop')
products = client[:products]
```

## Create - Inserting Documents

```ruby
# Insert a single document
result = products.insert_one({
  name:     'Laptop',
  price:    999.99,
  category: 'electronics',
  in_stock: true,
  created_at: Time.now
})

puts result.inserted_id  # BSON::ObjectId

# Insert multiple documents
result = products.insert_many([
  { name: 'Mouse', price: 29.99, category: 'electronics' },
  { name: 'Desk',  price: 199.00, category: 'furniture'  }
])
puts result.inserted_count  # => 2
```

## Read - Querying Documents

```ruby
# Find all documents
products.find.each { |doc| puts doc['name'] }

# Filter by field value
electronics = products.find(category: 'electronics').to_a

# Comparison operators
cheap = products.find(price: { '$lt' => 50 }).to_a

# Projection - return only specific fields
names = products.find({}, projection: { name: 1, price: 1, _id: 0 }).to_a

# Sort, skip, limit
page = products.find.sort(price: 1).skip(10).limit(10).to_a
```

## Update - Modifying Documents

```ruby
# Update a single document
products.update_one(
  { name: 'Laptop' },
  { '$set' => { price: 899.99, in_stock: false } }
)

# Update many documents
products.update_many(
  { category: 'electronics' },
  { '$inc' => { views: 1 } }
)

# Upsert - insert if not found
products.update_one(
  { name: 'Keyboard' },
  { '$setOnInsert' => { price: 49.99, category: 'electronics' } },
  upsert: true
)

# Replace an entire document
products.replace_one(
  { name: 'Mouse' },
  { name: 'Wireless Mouse', price: 39.99, category: 'electronics' }
)
```

## Delete - Removing Documents

```ruby
# Delete a single document
products.delete_one(name: 'Desk')

# Delete multiple documents
result = products.delete_many(category: 'furniture')
puts result.deleted_count

# Find and delete (returns the deleted document)
deleted = products.find_one_and_delete(name: 'Laptop')
puts deleted['price']
```

## Find and Modify in One Step

```ruby
# find_one_and_update returns the document before or after modification
updated = products.find_one_and_update(
  { name: 'Mouse' },
  { '$set' => { price: 34.99 } },
  return_document: :after
)
puts updated['price']  # => 34.99
```

## Summary

The MongoDB Ruby driver provides `insert_one`, `insert_many`, `find`, `update_one`, `update_many`, `replace_one`, `delete_one`, `delete_many`, and atomic find-and-modify methods. These map directly to MongoDB's own operations, making it straightforward to move between the MongoDB shell and Ruby code. Using update operators like `$set` and `$inc` lets you modify documents without replacing them entirely.
