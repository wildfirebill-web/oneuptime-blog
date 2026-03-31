# How to Write Integration Tests with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Integration Test, Testing, Node.js, Jest

Description: Learn how to write integration tests that run against a real MongoDB instance to verify that your queries, indexes, and schema validation work correctly end-to-end.

---

## Why Integration Tests?

Unit tests with mocks verify logic but cannot catch query bugs, missing indexes, or schema validation errors. Integration tests run against a real MongoDB instance and verify that your code works correctly with actual database behavior. They are essential for testing aggregation pipelines, transactions, and schema validation rules.

## Setup: Using jest-environment-node with a Real MongoDB

For integration tests, use either a local MongoDB instance, Docker, or `mongodb-memory-server`. This example uses a test MongoDB URI from the environment:

```bash
npm install --save-dev jest mongodb
```

```javascript
// jest.integration.config.js
module.exports = {
  testMatch: ['**/*.integration.test.js'],
  testTimeout: 30000,
  globalSetup: './tests/setup.js',
  globalTeardown: './tests/teardown.js',
};
```

## Global Setup and Teardown

```javascript
// tests/setup.js
const { MongoClient } = require('mongodb');

module.exports = async () => {
  const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';
  const client = new MongoClient(uri);
  await client.connect();
  global.__MONGO_CLIENT__ = client;
  await client.db('test').dropDatabase(); // start with clean slate
};
```

```javascript
// tests/teardown.js
module.exports = async () => {
  await global.__MONGO_CLIENT__.close();
};
```

## Writing Integration Tests

```javascript
// tests/userRepository.integration.test.js
const { MongoClient, ObjectId } = require('mongodb');

let client;
let db;
let users;

beforeAll(async () => {
  client = new MongoClient(process.env.MONGODB_URI || 'mongodb://localhost:27017');
  await client.connect();
  db = client.db('test');
  users = db.collection('users');
  // Create the index under test
  await users.createIndex({ email: 1 }, { unique: true });
});

afterAll(async () => {
  await client.close();
});

beforeEach(async () => {
  await users.deleteMany({});
});

describe('User collection integration', () => {
  it('enforces unique email constraint', async () => {
    await users.insertOne({ email: 'alice@example.com', name: 'Alice' });
    await expect(
      users.insertOne({ email: 'alice@example.com', name: 'Alice2' })
    ).rejects.toMatchObject({ code: 11000 });
  });

  it('finds users by email case-insensitively using collation', async () => {
    await users.insertOne({ email: 'Bob@example.com', name: 'Bob' });
    const result = await users.findOne(
      { email: 'bob@example.com' },
      { collation: { locale: 'en', strength: 2 } }
    );
    expect(result).not.toBeNull();
    expect(result.name).toBe('Bob');
  });

  it('returns null for a non-existent user', async () => {
    const result = await users.findOne({ _id: new ObjectId() });
    expect(result).toBeNull();
  });

  it('counts documents correctly after batch insert', async () => {
    await users.insertMany([
      { email: 'a@test.com' },
      { email: 'b@test.com' },
      { email: 'c@test.com' },
    ]);
    const count = await users.countDocuments();
    expect(count).toBe(3);
  });
});
```

## Running Integration Tests

```bash
# Run with a local MongoDB
MONGODB_URI=mongodb://localhost:27017 npx jest --config jest.integration.config.js

# Run with Docker MongoDB
docker run -d -p 27017:27017 mongo:7
npx jest --config jest.integration.config.js
```

## What to Test in Integration Tests

```text
- Unique index violations (code 11000)
- Schema validation enforcement (code 121)
- Aggregation pipeline results with real data
- Transaction atomicity (writes committed or rolled back)
- Index usage via explain() plans
- Change stream event delivery
```

## Summary

Integration tests complement unit tests by verifying that MongoDB-specific behavior - indexes, schema validation, aggregations, and transactions - works correctly. Use `beforeEach` with `deleteMany({})` to keep tests isolated, and always create the same indexes in your test setup as you have in production to catch missing-index bugs early.
