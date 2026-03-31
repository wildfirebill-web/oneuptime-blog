# How to Use MySQL with PHP MySQLi

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, PHP, MySQLi, Driver, Extension

Description: Learn how to use PHP's MySQLi extension to connect, query, and manage MySQL databases with both procedural and object-oriented interfaces.

---

## Overview

MySQLi ("MySQL Improved") is PHP's dedicated MySQL extension that replaced the old `mysql_*` functions. It supports both procedural and object-oriented styles and provides features like prepared statements, transactions, and multi-queries. Unlike PDO, MySQLi is MySQL-specific but includes some MySQL-only features.

## Enabling the Extension

MySQLi is bundled with PHP. Verify it is enabled:

```bash
php -m | grep mysqli
```

If not listed, enable `extension=mysqli` in `php.ini` and restart the web server.

## Connecting - Object-Oriented Style

```php
<?php
$host     = 'localhost';
$user     = 'app_user';
$password = 'secret';
$database = 'shop';

$mysqli = new mysqli($host, $user, $password, $database);

if ($mysqli->connect_error) {
    die('Connection failed: ' . $mysqli->connect_error);
}

// Set character set immediately after connecting
$mysqli->set_charset('utf8mb4');
echo 'Connected: MySQL ' . $mysqli->server_info;
```

## Connecting - Procedural Style

```php
<?php
$mysqli = mysqli_connect('localhost', 'app_user', 'secret', 'shop');
if (!$mysqli) {
    die('Connection failed: ' . mysqli_connect_error());
}
mysqli_set_charset($mysqli, 'utf8mb4');
```

## Running Simple Queries

```php
$result = $mysqli->query("SELECT id, name, price FROM products WHERE price < 50");

while ($row = $result->fetch_assoc()) {
    echo $row['name'] . ': $' . $row['price'] . PHP_EOL;
}

$result->free();
```

## Prepared Statements

Always use prepared statements for user-supplied data:

```php
$stmt = $mysqli->prepare(
    "SELECT id, name FROM users WHERE email = ? AND status = ?"
);
$stmt->bind_param('ss', $email, $status);

$email  = 'alice@example.com';
$status = 'active';
$stmt->execute();

$result = $stmt->get_result();
while ($row = $result->fetch_assoc()) {
    print_r($row);
}
$stmt->close();
```

## Inserting Data

```php
$stmt = $mysqli->prepare(
    "INSERT INTO orders (customer_id, total) VALUES (?, ?)"
);
$stmt->bind_param('id', $customerId, $total);

$customerId = 42;
$total      = 99.99;
$stmt->execute();

echo 'Inserted order ID: ' . $stmt->insert_id;
$stmt->close();
```

## Transactions

```php
$mysqli->begin_transaction();

try {
    $mysqli->query("UPDATE inventory SET stock = stock - 1 WHERE id = 7");
    $mysqli->query("INSERT INTO orders (product_id, qty) VALUES (7, 1)");
    $mysqli->commit();
} catch (Exception $e) {
    $mysqli->rollback();
    throw $e;
}
```

## Closing the Connection

```php
$mysqli->close();
```

## Summary

MySQLi is a robust, MySQL-specific PHP extension with both procedural and OO interfaces. Always call `set_charset('utf8mb4')` immediately after connecting, use prepared statements with `bind_param()` for all user input, and wrap multi-step writes in transactions. For projects that need database portability, consider PDO instead.
