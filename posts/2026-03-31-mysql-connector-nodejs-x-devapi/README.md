# How to Use MySQL Connector/Node.js with X DevAPI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Node.js, X DevAPI, Connector, JavaScript

Description: Learn how to use the official MySQL Connector/Node.js with the X DevAPI to connect, query, and manage MySQL data from Node.js applications.

---

## What is MySQL Connector/Node.js?

MySQL Connector/Node.js (`@mysql/xdevapi`) is the official MySQL driver for Node.js that communicates over the X Protocol (port 33060). Unlike `mysql2` or `mysql` packages that use the classic protocol, this connector uses the X DevAPI for both document store and relational table operations. It supports async/await natively and connection pooling.

## Installation

```bash
npm install @mysql/xdevapi
```

## Basic Connection

```javascript
const mysqlx = require('@mysql/xdevapi');

async function main() {
  const session = await mysqlx.getSession({
    host: '127.0.0.1',
    port: 33060,
    user: 'root',
    password: 'secret',
    schema: 'mydb'
  });

  console.log('Connected:', session.inspect());
  await session.close();
}

main().catch(console.error);
```

## Connection Using a URI

```javascript
const session = await mysqlx.getSession('mysqlx://root:secret@127.0.0.1:33060/mydb');
```

## Connection Pooling

For production applications, use a connection pool to avoid the overhead of creating new sessions per request:

```javascript
const client = mysqlx.getClient(
  {
    host: '127.0.0.1',
    port: 33060,
    user: 'root',
    password: 'secret',
    schema: 'mydb'
  },
  {
    pooling: {
      enabled: true,
      maxSize: 25,
      maxIdleTime: 30000,
      queueTimeout: 10000
    }
  }
);

// Acquire a session from the pool
const session = await client.getSession();
// ... use session ...
await session.close(); // returns to pool

// Shutdown pool
await client.close();
```

## Document Store Operations

```javascript
const schema = session.getDefaultSchema();
const orders = schema.getCollection('orders');

// Add
await orders.add({ customerId: 1, total: 75.50, status: 'pending' }).execute();

// Find
const result = await orders.find('status = :s')
  .bind('s', 'pending')
  .sort(['total DESC'])
  .limit(10)
  .execute();

const docs = result.fetchAll();

// Modify
await orders.modify('customerId = :id')
  .set('status', 'confirmed')
  .bind('id', 1)
  .execute();

// Remove
await orders.remove('status = :s').bind('s', 'cancelled').execute();
```

## Relational Table Operations

```javascript
const customers = schema.getTable('customers');

// Insert
await customers.insert(['name', 'email'])
  .values(['Eve', 'eve@example.com'])
  .execute();

// Select
const rows = await customers.select(['id', 'name'])
  .where('email LIKE :pattern')
  .bind('pattern', '%@example.com')
  .execute();

rows.fetchAll().forEach(row => console.log(row[0], row[1]));
```

## Running SQL Directly

```javascript
const result = await session.sql(
  'SELECT id, name FROM customers WHERE tier = ? LIMIT 5'
).bind('gold').execute();

result.fetchAll().forEach(row => console.log(row));
```

## Error Handling

```javascript
try {
  const session = await mysqlx.getSession({ host: '127.0.0.1', port: 33060, user: 'root', password: 'wrong' });
} catch (err) {
  console.error('Connection failed:', err.message);
}
```

## Summary

MySQL Connector/Node.js with X DevAPI provides a modern, promise-based interface for MySQL from Node.js. Use `getClient` with pooling enabled for production workloads, prefer parameter binding over string interpolation for all queries, and use `session.sql()` for complex queries that benefit from raw SQL. The connector handles both document and relational access through a single consistent API.
