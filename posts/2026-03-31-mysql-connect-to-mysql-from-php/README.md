# How to Connect to MySQL from PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, PHP, PDO, Connection, Driver

Description: Learn how to connect to a MySQL database from PHP using PDO and MySQLi, with best practices for security and character set configuration.

---

## Overview

PHP offers two primary interfaces for MySQL connectivity:

| Interface | Type | Best For |
|-----------|------|----------|
| PDO | Multi-database abstraction | Portability across databases |
| MySQLi | MySQL-specific | MySQL-only projects, multi-queries |

Both support prepared statements, transactions, and connection options. This guide shows both.

## Connecting with PDO

PDO (PHP Data Objects) is the recommended approach for new PHP applications because it abstracts the database layer and supports multiple backends.

```php
<?php
$dsn = 'mysql:host=localhost;port=3306;dbname=shop;charset=utf8mb4';

$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
];

try {
    $pdo = new PDO($dsn, 'app_user', 'secret', $options);
    echo 'Connected successfully';
} catch (PDOException $e) {
    die('Connection failed: ' . $e->getMessage());
}
```

Note the `charset=utf8mb4` in the DSN - this sets the character encoding without a separate `SET NAMES` call.

## Querying with PDO

```php
// Simple query
$stmt = $pdo->query("SELECT VERSION() AS version");
$row = $stmt->fetch();
echo 'MySQL ' . $row['version'];

// Parameterized query
$stmt = $pdo->prepare("SELECT id, name FROM products WHERE price < ?");
$stmt->execute([50.00]);
$products = $stmt->fetchAll();
foreach ($products as $p) {
    echo $p['name'] . PHP_EOL;
}
```

## Connecting with MySQLi

```php
<?php
$mysqli = new mysqli('localhost', 'app_user', 'secret', 'shop');

if ($mysqli->connect_error) {
    die('Connection failed: ' . $mysqli->connect_error);
}

$mysqli->set_charset('utf8mb4');
echo 'Connected: MySQL ' . $mysqli->server_info;
```

## Loading Credentials from the Environment

```php
<?php
$dsn = sprintf(
    'mysql:host=%s;port=%s;dbname=%s;charset=utf8mb4',
    getenv('DB_HOST'),
    getenv('DB_PORT') ?: '3306',
    getenv('DB_NAME')
);

$pdo = new PDO($dsn, getenv('DB_USER'), getenv('DB_PASSWORD'), [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
]);
```

## Inserting Data with PDO

```php
$stmt = $pdo->prepare(
    "INSERT INTO orders (customer_id, total, created_at) VALUES (?, ?, NOW())"
);
$stmt->execute([42, 99.99]);
$orderId = $pdo->lastInsertId();
echo "Created order: $orderId";
```

## Using Transactions

```php
$pdo->beginTransaction();
try {
    $pdo->prepare("UPDATE stock SET qty = qty - 1 WHERE id = ?")->execute([7]);
    $pdo->prepare("INSERT INTO order_items (order_id, item_id) VALUES (?, ?)")->execute([100, 7]);
    $pdo->commit();
} catch (Exception $e) {
    $pdo->rollBack();
    throw $e;
}
```

## Summary

For new PHP projects use PDO with `charset=utf8mb4` in the DSN and `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION` for proper error handling. Use MySQLi when you need MySQL-specific features such as multi-queries or the `LOAD DATA` call. Always use prepared statements and load credentials from the environment.
