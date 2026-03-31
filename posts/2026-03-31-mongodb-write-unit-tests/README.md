# How to Write Unit Tests for MongoDB Operations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Unit Test, Jest, Testing, Node.js

Description: Learn how to write effective unit tests for MongoDB operations by mocking the driver to test business logic in isolation without a real database.

---

## Why Unit Test MongoDB Operations?

Unit tests for MongoDB operations verify business logic in isolation - without spinning up a real database. They run fast, require no external infrastructure, and give precise feedback about which code path broke. The key is to mock the MongoDB driver so tests focus on the logic around database calls, not the database itself.

## Project Setup

```bash
npm install --save-dev jest @jest/globals
npm install mongodb
```

## The Service Under Test

```javascript
// src/userService.js
class UserService {
  constructor(db) {
    this.collection = db.collection('users');
  }

  async createUser(userData) {
    if (!userData.email) throw new Error('Email is required');
    const existing = await this.collection.findOne({ email: userData.email });
    if (existing) throw new Error('User already exists');
    const result = await this.collection.insertOne({
      ...userData,
      createdAt: new Date(),
    });
    return result.insertedId;
  }

  async getUserById(id) {
    const user = await this.collection.findOne({ _id: id });
    if (!user) throw new Error('User not found');
    return user;
  }
}

module.exports = UserService;
```

## Writing Unit Tests with Jest

```javascript
// tests/userService.test.js
const { ObjectId } = require('mongodb');
const UserService = require('../src/userService');

describe('UserService', () => {
  let mockCollection;
  let mockDb;
  let service;

  beforeEach(() => {
    // Create mock collection with jest.fn() for each method
    mockCollection = {
      findOne: jest.fn(),
      insertOne: jest.fn(),
    };
    mockDb = {
      collection: jest.fn().mockReturnValue(mockCollection),
    };
    service = new UserService(mockDb);
  });

  describe('createUser', () => {
    it('throws if email is missing', async () => {
      await expect(service.createUser({ name: 'Alice' }))
        .rejects.toThrow('Email is required');
    });

    it('throws if user already exists', async () => {
      mockCollection.findOne.mockResolvedValue({ email: 'alice@example.com' });
      await expect(service.createUser({ email: 'alice@example.com' }))
        .rejects.toThrow('User already exists');
    });

    it('creates a new user and returns the insertedId', async () => {
      const id = new ObjectId();
      mockCollection.findOne.mockResolvedValue(null);
      mockCollection.insertOne.mockResolvedValue({ insertedId: id });

      const result = await service.createUser({ email: 'bob@example.com', name: 'Bob' });
      expect(result).toEqual(id);
      expect(mockCollection.insertOne).toHaveBeenCalledWith(
        expect.objectContaining({ email: 'bob@example.com', createdAt: expect.any(Date) })
      );
    });
  });

  describe('getUserById', () => {
    it('throws if user not found', async () => {
      mockCollection.findOne.mockResolvedValue(null);
      await expect(service.getUserById(new ObjectId()))
        .rejects.toThrow('User not found');
    });

    it('returns the user document', async () => {
      const user = { _id: new ObjectId(), email: 'alice@example.com' };
      mockCollection.findOne.mockResolvedValue(user);
      const result = await service.getUserById(user._id);
      expect(result).toEqual(user);
    });
  });
});
```

## Running Tests

```bash
npx jest --coverage
```

## Best Practices

```text
- Inject the db/collection as a dependency for easy mocking
- Test each code path: success, not-found, already-exists, invalid-input
- Use expect.objectContaining() to check partial documents
- Reset mocks with beforeEach() to prevent test pollution
- Test that insertOne/updateOne are called with the correct arguments
```

## Summary

Unit tests for MongoDB operations mock the driver and focus purely on business logic. By injecting the `db` object as a dependency and using `jest.fn()` for each collection method, you can test all code paths quickly without any database infrastructure. This gives you fast feedback during development and catches regressions in your service layer logic.
