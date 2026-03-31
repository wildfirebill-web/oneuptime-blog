# How to Create Test Fixtures for MongoDB in CI/CD

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Testing, Fixture, CI/CD, Automation

Description: Learn how to create reusable test fixtures for MongoDB in CI/CD pipelines using seed scripts, factory patterns, and snapshot-based approaches.

---

## Introduction

Test fixtures provide consistent, repeatable data for your tests. In MongoDB CI/CD workflows, well-designed fixtures mean your tests are deterministic and don't depend on production data. This guide covers creating fixtures using JSON files, JavaScript seed scripts, and factory patterns in Node.js.

## JSON-Based Fixtures

Store fixture data as JSON files in your repository:

```json
[
  {
    "_id": { "$oid": "507f1f77bcf86cd799439011" },
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "role": "admin",
    "createdAt": { "$date": "2025-01-01T00:00:00Z" }
  },
  {
    "_id": { "$oid": "507f1f77bcf86cd799439012" },
    "name": "Bob Smith",
    "email": "bob@example.com",
    "role": "user",
    "createdAt": { "$date": "2025-01-02T00:00:00Z" }
  }
]
```

Load fixtures with `mongoimport`:

```bash
mongoimport \
  --uri="mongodb://localhost:27017/testdb" \
  --collection=users \
  --file=fixtures/users.json \
  --jsonArray \
  --drop
```

## JavaScript Seed Script

```javascript
// fixtures/seed.js
const { MongoClient } = require("mongodb")

const fixtures = {
  users: [
    { name: "Alice", email: "alice@test.com", role: "admin" },
    { name: "Bob", email: "bob@test.com", role: "user" }
  ],
  products: [
    { name: "Widget", price: 9.99, stock: 100, category: "hardware" },
    { name: "Gadget", price: 24.99, stock: 50, category: "electronics" }
  ]
}

async function seed(uri = "mongodb://localhost:27017/testdb") {
  const client = new MongoClient(uri)
  await client.connect()
  const db = client.db()

  for (const [collection, docs] of Object.entries(fixtures)) {
    await db.collection(collection).drop().catch(() => {})
    await db.collection(collection).insertMany(docs)
    console.log(`Seeded ${docs.length} documents into ${collection}`)
  }

  await client.close()
}

seed(process.env.MONGODB_URI).catch(console.error)
```

## Factory Pattern with Faker

Generate realistic fixture data programmatically:

```javascript
// fixtures/factory.js
const { faker } = require("@faker-js/faker")

function createUser(overrides = {}) {
  return {
    name: faker.person.fullName(),
    email: faker.internet.email(),
    role: "user",
    createdAt: new Date(),
    ...overrides
  }
}

function createProduct(overrides = {}) {
  return {
    name: faker.commerce.productName(),
    price: parseFloat(faker.commerce.price()),
    stock: faker.number.int({ min: 0, max: 500 }),
    category: faker.commerce.department(),
    ...overrides
  }
}

async function seedDatabase(db, counts = {}) {
  const userCount = counts.users || 10
  const productCount = counts.products || 20

  await db.collection("users").insertMany(
    Array.from({ length: userCount }, () => createUser())
  )
  await db.collection("products").insertMany(
    Array.from({ length: productCount }, () => createProduct())
  )
}

module.exports = { createUser, createProduct, seedDatabase }
```

## pytest Fixture Pattern

```python
# tests/conftest.py
import pytest
from pymongo import MongoClient
import os

FIXTURES = {
    "users": [
        {"name": "Alice", "email": "alice@test.com", "role": "admin"},
        {"name": "Bob", "email": "bob@test.com", "role": "user"},
    ]
}

@pytest.fixture(scope="module")
def db():
    client = MongoClient(os.environ["MONGODB_URI"])
    database = client.testdb
    for collection, docs in FIXTURES.items():
        database[collection].drop()
        database[collection].insert_many(docs)
    yield database
    client.close()
```

## CI/CD Integration

```yaml
# GitHub Actions step
- name: Load fixtures
  run: node fixtures/seed.js
  env:
    MONGODB_URI: mongodb://localhost:27017/testdb
```

## Summary

Effective MongoDB test fixtures use a combination of static JSON files for known entity IDs (useful for referential integrity), seed scripts for setup and teardown automation, and factory functions for generating realistic bulk data. In CI/CD pipelines, run fixture loading as a `before_script` or setup step, always drop and recreate collections to ensure clean state, and store fixtures in version control alongside your application code.
