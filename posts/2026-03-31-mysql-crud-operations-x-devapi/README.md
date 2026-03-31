# How to Use CRUD Operations with MySQL X DevAPI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, CRUD, X DevAPI, Collection, Table

Description: Learn how to perform Create, Read, Update, and Delete operations using the MySQL X DevAPI for both document collections and relational tables.

---

## Overview

The MySQL X DevAPI provides a CRUD interface for both document collections and SQL tables. Operations are expressed as method chains rather than raw SQL strings, making them composable and easier to work with in application code. This guide covers CRUD for collections (document store) and table CRUD for relational data.

## Setup

Install the Node.js connector:

```bash
npm install @mysql/xdevapi
```

Create a session:

```javascript
const mysqlx = require('@mysql/xdevapi');

async function getSession() {
  return mysqlx.getSession({
    host: '127.0.0.1',
    port: 33060,
    user: 'root',
    password: 'secret',
    schema: 'mydb'
  });
}
```

## Collection CRUD

### Create (Add Documents)

```javascript
const session = await getSession();
const schema = session.getDefaultSchema();
const orders = schema.getCollection('orders');

await orders.add([
  { customer_id: 1, items: [{ sku: 'A1', qty: 2 }], total: 39.98, status: 'pending' },
  { customer_id: 2, items: [{ sku: 'B3', qty: 1 }], total: 99.00, status: 'pending' }
]).execute();
```

### Read (Find Documents)

```javascript
// All pending orders
const result = await orders.find('status = :s')
  .bind('s', 'pending')
  .fields(['customer_id', 'total', 'status'])
  .sort(['total DESC'])
  .limit(10)
  .execute();

const docs = result.fetchAll();
docs.forEach(doc => console.log(doc));
```

### Update (Modify Documents)

```javascript
await orders.modify('status = :old')
  .set('status', 'shipped')
  .bind('old', 'pending')
  .execute();
```

### Delete (Remove Documents)

```javascript
await orders.remove('status = :s AND total < :t')
  .bind('s', 'cancelled')
  .bind('t', 5)
  .execute();
```

## Table CRUD

### Create (Insert Rows)

```javascript
const customers = schema.getTable('customers');

await customers.insert(['name', 'email', 'tier'])
  .values(['Alice', 'alice@example.com', 'gold'])
  .values(['Bob',   'bob@example.com',   'silver'])
  .execute();
```

### Read (Select Rows)

```javascript
const result = await customers.select(['id', 'name', 'email'])
  .where('tier = :tier')
  .bind('tier', 'gold')
  .orderBy(['name ASC'])
  .execute();

result.fetchAll().forEach(row => console.log(row));
```

### Update (Update Rows)

```javascript
await customers.update()
  .set('tier', 'platinum')
  .where('id = :id')
  .bind('id', 42)
  .execute();
```

### Delete (Delete Rows)

```javascript
await customers.delete()
  .where('tier = :t AND last_order < :d')
  .bind('t', 'inactive')
  .bind('d', '2023-01-01')
  .execute();
```

## Error Handling

```javascript
try {
  await orders.add({ customer_id: null, total: -1 }).execute();
} catch (err) {
  console.error('Insert failed:', err.message);
} finally {
  await session.close();
}
```

## Summary

The MySQL X DevAPI CRUD interface provides a clean, method-chaining approach to data access for both collections and tables. Use `add/find/modify/remove` for document collections and `insert/select/update/delete` for relational tables. Always bind parameters using `.bind()` rather than string concatenation to prevent injection vulnerabilities, and close sessions when done to return connections to the pool.
