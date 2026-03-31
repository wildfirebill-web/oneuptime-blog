# How to Implement Connection Pooling for MySQL in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Connection Pool, Node.js, mysql2, Performance

Description: Learn how to implement MySQL connection pooling in Node.js using the mysql2 library's built-in pool for efficient concurrent database access.

---

## Introduction

Node.js is single-threaded but handles I/O asynchronously. MySQL connection pooling in Node.js ensures that multiple concurrent async operations share a fixed set of connections, avoiding the overhead of creating and tearing down connections for every query while also respecting MySQL's `max_connections` limit.

## Installing mysql2

```bash
npm install mysql2
```

## Creating a Connection Pool

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'localhost',
  port: 3306,
  user: 'root',
  password: 'password',
  database: 'mydb',
  charset: 'utf8mb4',
  waitForConnections: true,
  connectionLimit: 10,
  maxIdle: 5,
  idleTimeout: 60000,    // release idle connections after 60s
  queueLimit: 0,         // unlimited queue (use a number to cap backpressure)
  enableKeepAlive: true,
  keepAliveInitialDelay: 0,
});

module.exports = pool;
```

## Using the Pool

```javascript
const pool = require('./db');

// Simple query
async function getProducts(maxPrice) {
  const [rows] = await pool.query(
    'SELECT id, name, price, stock FROM products WHERE price <= ? AND stock > 0 ORDER BY price',
    [maxPrice]
  );
  return rows;
}

// Using a connection explicitly
async function transferStock(fromId, toId, qty) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();
    await conn.execute('UPDATE products SET stock = stock - ? WHERE id = ?', [qty, fromId]);
    await conn.execute('UPDATE products SET stock = stock + ? WHERE id = ?', [qty, toId]);
    await conn.commit();
  } catch (err) {
    await conn.rollback();
    throw err;
  } finally {
    conn.release(); // returns connection to pool
  }
}
```

## Pool Events and Error Handling

```javascript
const mysql = require('mysql2');

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb',
  connectionLimit: 10,
});

pool.on('connection', (conn) => {
  console.log(`New connection established, id: ${conn.threadId}`);
});

pool.on('acquire', (conn) => {
  console.log(`Connection ${conn.threadId} acquired`);
});

pool.on('release', (conn) => {
  console.log(`Connection ${conn.threadId} released`);
});

pool.on('enqueue', () => {
  console.log('Waiting for available connection in pool');
});
```

## Using the Pool with Prepared Statements

```javascript
async function insertProduct(name, price, stock, categoryId) {
  const [result] = await pool.execute(
    'INSERT INTO products (name, price, stock, category_id) VALUES (?, ?, ?, ?)',
    [name, price, stock, categoryId]
  );
  return result.insertId;
}
```

`pool.execute` uses server-side prepared statements for improved performance and SQL injection protection.

## Integrating with Express

```javascript
const express = require('express');
const pool = require('./db');

const app = express();
app.use(express.json());

app.get('/products', async (req, res) => {
  try {
    const maxPrice = parseFloat(req.query.maxPrice) || 10000;
    const [rows] = await pool.query(
      'SELECT id, name, price FROM products WHERE price <= ? AND stock > 0',
      [maxPrice]
    );
    res.json(rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

app.listen(3000);
```

## Monitoring Pool Status

```javascript
// mysql2 does not expose pool stats directly, use this workaround
const poolStatus = {
  all: pool.pool._allConnections.length,
  free: pool.pool._freeConnections.length,
  queue: pool.pool._connectionQueue.length,
};
console.log(poolStatus);
```

## Summary

The `mysql2` library's `createPool` is the recommended approach for MySQL connection management in Node.js. Set `connectionLimit` based on your MySQL server's `max_connections` and the number of Node.js processes running, enable `enableKeepAlive` to prevent stale connections, and always call `conn.release()` in a `finally` block when using explicit connections. Use `pool.execute()` for prepared statements and `pool.query()` for one-off queries.
