# How to Use State Transactions with Multiple Keys in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Transaction, Atomicity, Microservice

Description: Learn how to use Dapr state transactions to perform atomic multi-key operations - applying multiple saves and deletes as a single all-or-nothing unit of work.

---

## Why Use State Transactions?

In distributed systems, updating multiple related keys one at a time risks partial failure. Dapr state transactions let you bundle multiple upsert and delete operations into a single atomic call, ensuring all changes apply together or none do.

## Transaction API

Dapr exposes a transaction endpoint that accepts a list of operations:

```javascript
const { DaprClient } = require('@dapr/dapr');
const client = new DaprClient();

async function transferFunds(fromAccountId, toAccountId, amount) {
  const fromKey = `account:${fromAccountId}`;
  const toKey = `account:${toAccountId}`;

  // Read both accounts first
  const from = await client.state.get('state-store', fromKey);
  const to = await client.state.get('state-store', toKey);

  if (from.balance < amount) {
    throw new Error('Insufficient funds');
  }

  // Execute as a transaction
  await client.state.transaction('state-store', [
    {
      operation: 'upsert',
      request: {
        key: fromKey,
        value: { ...from, balance: from.balance - amount }
      }
    },
    {
      operation: 'upsert',
      request: {
        key: toKey,
        value: { ...to, balance: to.balance + amount }
      }
    },
    {
      operation: 'upsert',
      request: {
        key: `transfer:${Date.now()}`,
        value: { fromAccountId, toAccountId, amount, status: 'completed' }
      }
    }
  ]);
}
```

## Using the HTTP API Directly

If you prefer raw HTTP calls:

```bash
curl -X POST http://localhost:3500/v1.0/state/state-store/transaction \
  -H "Content-Type: application/json" \
  -d '{
    "operations": [
      {
        "operation": "upsert",
        "request": {
          "key": "account:alice",
          "value": { "balance": 900 }
        }
      },
      {
        "operation": "upsert",
        "request": {
          "key": "account:bob",
          "value": { "balance": 1100 }
        }
      },
      {
        "operation": "delete",
        "request": {
          "key": "pending:transfer:xyz"
        }
      }
    ]
  }'
```

## Transactional Order Processing

```javascript
async function processOrder(orderId, cartId, userId) {
  const cart = await client.state.get('state-store', `cart:${cartId}`);

  await client.state.transaction('state-store', [
    // Create the order
    {
      operation: 'upsert',
      request: {
        key: `order:${orderId}`,
        value: { id: orderId, userId, items: cart.items, status: 'processing' }
      }
    },
    // Clear the cart
    {
      operation: 'delete',
      request: { key: `cart:${cartId}` }
    },
    // Add to user order history
    {
      operation: 'upsert',
      request: {
        key: `user:${userId}:latest-order`,
        value: orderId
      }
    }
  ]);
}
```

## Supported State Stores

Not all state stores support transactions. Confirmed transactional stores include:

- Redis
- Azure Cosmos DB
- PostgreSQL
- MongoDB
- MySQL

```bash
# Check if your store supports transactions
dapr run --app-id check --log-level debug -- sleep 2 2>&1 | grep "transaction"
```

## Summary

Dapr state transactions provide atomicity for multi-key operations with a clean, portable API. By batching related upserts and deletes into a single transaction, you eliminate partial-update bugs and keep complex state changes consistent without managing database transactions directly in your application code.
