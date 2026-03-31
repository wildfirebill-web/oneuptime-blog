# How to Mock MongoDB Operations with Jest

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Jest, Mock, Testing, Node.js

Description: Learn how to mock MongoDB driver operations with Jest to write fast unit tests that verify business logic without connecting to a real database.

---

## Why Mock MongoDB in Unit Tests?

Mocking MongoDB operations in unit tests lets you verify your application logic without connecting to a database. Tests run in milliseconds, work offline, and give precise control over what the database "returns" - including error conditions that are hard to reproduce with a real server.

## Manual Mock Approach

The most flexible approach is to create a mock object with `jest.fn()` for each collection method:

```javascript
// tests/helpers/mockMongo.js
const createMockCollection = () => ({
  findOne: jest.fn(),
  find: jest.fn().mockReturnValue({
    toArray: jest.fn().mockResolvedValue([]),
    sort: jest.fn().mockReturnThis(),
    limit: jest.fn().mockReturnThis(),
    skip: jest.fn().mockReturnThis(),
  }),
  insertOne: jest.fn(),
  insertMany: jest.fn(),
  updateOne: jest.fn(),
  updateMany: jest.fn(),
  deleteOne: jest.fn(),
  deleteMany: jest.fn(),
  countDocuments: jest.fn(),
  aggregate: jest.fn().mockReturnValue({
    toArray: jest.fn().mockResolvedValue([]),
  }),
});

const createMockDb = (collections = {}) => ({
  collection: jest.fn((name) => collections[name] || createMockCollection()),
});

module.exports = { createMockCollection, createMockDb };
```

## Using the Mock in Tests

```javascript
// tests/orderService.test.js
const { createMockDb, createMockCollection } = require('./helpers/mockMongo');
const OrderService = require('../src/orderService');

describe('OrderService', () => {
  let ordersCollection;
  let inventoryCollection;
  let db;
  let service;

  beforeEach(() => {
    ordersCollection = createMockCollection();
    inventoryCollection = createMockCollection();
    db = createMockDb({
      orders: ordersCollection,
      inventory: inventoryCollection,
    });
    service = new OrderService(db);
  });

  it('places an order and decrements inventory', async () => {
    inventoryCollection.findOne.mockResolvedValue({ sku: 'W001', qty: 10 });
    ordersCollection.insertOne.mockResolvedValue({ insertedId: 'order-1' });
    inventoryCollection.updateOne.mockResolvedValue({ modifiedCount: 1 });

    const result = await service.placeOrder({ sku: 'W001', qty: 2 });
    expect(result).toBe('order-1');
    expect(inventoryCollection.updateOne).toHaveBeenCalledWith(
      { sku: 'W001' },
      { $inc: { qty: -2 } }
    );
  });

  it('throws if item is out of stock', async () => {
    inventoryCollection.findOne.mockResolvedValue({ sku: 'W001', qty: 0 });
    await expect(service.placeOrder({ sku: 'W001', qty: 1 }))
      .rejects.toThrow('Out of stock');
  });
});
```

## Mocking with jest.mock()

For automatic module mocking:

```javascript
// tests/userRepo.test.js
jest.mock('mongodb');

const { MongoClient } = require('mongodb');
const mockInsertOne = jest.fn().mockResolvedValue({ insertedId: 'abc' });
const mockFindOne = jest.fn().mockResolvedValue(null);

MongoClient.prototype.connect = jest.fn();
MongoClient.prototype.db = jest.fn().mockReturnValue({
  collection: jest.fn().mockReturnValue({
    insertOne: mockInsertOne,
    findOne: mockFindOne,
  }),
});
```

## Simulating Database Errors

```javascript
it('handles duplicate key error gracefully', async () => {
  const { MongoServerError } = require('mongodb');
  const error = new MongoServerError({ message: 'E11000 duplicate key' });
  error.code = 11000;
  ordersCollection.insertOne.mockRejectedValue(error);

  await expect(service.placeOrder({ sku: 'W001', qty: 1 }))
    .rejects.toThrow('already exists');
});
```

## Checking Call Arguments

```javascript
it('passes the correct filter to findOne', async () => {
  ordersCollection.findOne.mockResolvedValue({ _id: 'ord-1', status: 'pending' });
  await service.getOrder('ord-1');
  expect(ordersCollection.findOne).toHaveBeenCalledWith({ _id: 'ord-1' });
});
```

## Summary

Mock MongoDB operations with `jest.fn()` to test your service layer in complete isolation. Create reusable mock helpers for collections and databases, simulate success and error responses for each test case, and use `toHaveBeenCalledWith()` to verify that your code constructs the correct queries. Combine unit tests with integration tests using `mongodb-memory-server` for full coverage.
