# How to Connect to MySQL from Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Node.js, JavaScript, Driver, Connection

Description: Learn how to connect to a MySQL database from Node.js using the mysql2 driver and manage connections with pools for production applications.

---

## Overview

The most widely used MySQL driver for Node.js is `mysql2`, which supports promises, prepared statements, and connection pooling. The older `mysql` package is no longer actively maintained, so `mysql2` is the recommended choice.

## Installation

```bash
npm install mysql2
```

## Basic Connection

```javascript
const mysql = require('mysql2');

const connection = mysql.createConnection({
  host: 'localhost',
  port: 3306,
  user: 'app_user',
  password: 'secret',
  database: 'shop',
  charset: 'utf8mb4'
});

connection.connect((err) => {
  if (err) {
    console.error('Connection error:', err.message);
    process.exit(1);
  }
  console.log('Connected to MySQL');
});

connection.query('SELECT VERSION() AS version', (err, results) => {
  if (err) throw err;
  console.log('MySQL version:', results[0].version);
});

connection.end();
```

## Using Promises

`mysql2` ships with a promise wrapper. Use `mysql2/promise` for async/await syntax:

```javascript
const mysql = require('mysql2/promise');

async function main() {
  const connection = await mysql.createConnection({
    host: 'localhost',
    user: 'app_user',
    password: 'secret',
    database: 'shop',
    charset: 'utf8mb4'
  });

  const [rows] = await connection.execute(
    'SELECT id, name, price FROM products WHERE price < ?',
    [50.00]
  );

  console.log(rows);
  await connection.end();
}

main().catch(console.error);
```

## Connection Pooling for Production

Single connections are fine for scripts but production applications should use a pool:

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  charset: 'utf8mb4',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0
});

// Use pool.execute() - connections are acquired and released automatically
async function getProducts(maxPrice) {
  const [rows] = await pool.execute(
    'SELECT id, name, price FROM products WHERE price < ?',
    [maxPrice]
  );
  return rows;
}
```

## Inserting Data

```javascript
async function createOrder(customerId, total) {
  const [result] = await pool.execute(
    'INSERT INTO orders (customer_id, total, created_at) VALUES (?, ?, NOW())',
    [customerId, total]
  );
  return result.insertId;
}
```

## Using Environment Variables

```javascript
const pool = mysql.createPool({
  host:     process.env.DB_HOST     || 'localhost',
  user:     process.env.DB_USER     || 'root',
  password: process.env.DB_PASSWORD || '',
  database: process.env.DB_NAME,
  charset:  'utf8mb4'
});
```

## Summary

Use `mysql2` with the promise interface for modern Node.js MySQL connectivity. Single connections suit short scripts; use a pool (`createPool`) for web servers and long-running services. Always use `execute()` with `?` placeholders for parameterized queries, and load credentials from environment variables rather than hardcoding them.
