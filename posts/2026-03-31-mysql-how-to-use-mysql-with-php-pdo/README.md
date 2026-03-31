# How to Use MySQL with PHP PDO

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, PHP, PDO, Database Integration, Web Development

Description: Learn how to connect to and query MySQL from PHP using PDO with prepared statements, transactions, and error handling best practices.

---

## What is PHP PDO

PDO (PHP Data Objects) is a database access abstraction layer built into PHP. It provides a consistent interface to multiple database engines (MySQL, PostgreSQL, SQLite, etc.) and supports prepared statements, transactions, and error handling.

PDO with MySQL is the recommended approach over the older `mysql_*` functions or `mysqli`.

## Requirements

Ensure the PDO MySQL extension is enabled in `php.ini`:

```text
extension=pdo_mysql
```

Check if it's loaded:

```bash
php -m | grep pdo
```

## Connecting to MySQL

```php
<?php
$dsn = 'mysql:host=localhost;dbname=your_database;charset=utf8mb4';
$user = 'your_user';
$password = 'your_password';

$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    PDO::ATTR_EMULATE_PREPARES   => false,
];

try {
    $pdo = new PDO($dsn, $user, $password, $options);
    echo "Connected successfully\n";
} catch (PDOException $e) {
    die("Connection failed: " . $e->getMessage());
}
```

## Inserting Data with Prepared Statements

```php
<?php
$stmt = $pdo->prepare(
    'INSERT INTO employees (name, department, salary) VALUES (:name, :department, :salary)'
);

$stmt->execute([
    ':name'       => 'Alice Smith',
    ':department' => 'Engineering',
    ':salary'     => 95000.00
]);

echo "Inserted ID: " . $pdo->lastInsertId() . "\n";
```

Using positional placeholders:

```php
<?php
$stmt = $pdo->prepare('INSERT INTO employees (name, department, salary) VALUES (?, ?, ?)');
$stmt->execute(['Bob Jones', 'Marketing', 72000.00]);
```

## Querying Data

```php
<?php
// Fetch all rows
$stmt = $pdo->prepare('SELECT * FROM employees WHERE department = ?');
$stmt->execute(['Engineering']);
$employees = $stmt->fetchAll();

foreach ($employees as $emp) {
    echo $emp['name'] . ' - ' . $emp['salary'] . "\n";
}

// Fetch one row
$stmt = $pdo->prepare('SELECT * FROM employees WHERE id = ?');
$stmt->execute([1]);
$employee = $stmt->fetch();
echo $employee['name'];

// Fetch a single column
$stmt = $pdo->query('SELECT COUNT(*) FROM employees');
$count = $stmt->fetchColumn();
echo "Total employees: $count";
```

## Updating Records

```php
<?php
$stmt = $pdo->prepare('UPDATE employees SET salary = ? WHERE id = ?');
$stmt->execute([105000.00, 1]);
echo "Rows updated: " . $stmt->rowCount() . "\n";
```

## Deleting Records

```php
<?php
$stmt = $pdo->prepare('DELETE FROM employees WHERE id = ?');
$stmt->execute([5]);
echo "Rows deleted: " . $stmt->rowCount() . "\n";
```

## Using Transactions

```php
<?php
try {
    $pdo->beginTransaction();

    $pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
        ->execute([500.00, 1]);

    $pdo->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?')
        ->execute([500.00, 2]);

    $pdo->commit();
    echo "Transaction committed\n";
} catch (PDOException $e) {
    $pdo->rollBack();
    echo "Transaction rolled back: " . $e->getMessage() . "\n";
}
```

## Fetch Modes

```php
<?php
// Fetch as object
$stmt = $pdo->query('SELECT * FROM employees');
while ($emp = $stmt->fetchObject()) {
    echo $emp->name . "\n";
}

// Fetch into a class
$stmt = $pdo->query('SELECT * FROM employees');
$stmt->setFetchMode(PDO::FETCH_CLASS, 'Employee');
$employees = $stmt->fetchAll();
```

## Using a Reusable Database Connection

```php
<?php
class Database {
    private static ?PDO $instance = null;

    public static function getInstance(): PDO {
        if (self::$instance === null) {
            $dsn = 'mysql:host=localhost;dbname=your_database;charset=utf8mb4';
            self::$instance = new PDO($dsn, 'user', 'password', [
                PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
                PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            ]);
        }
        return self::$instance;
    }
}

$pdo = Database::getInstance();
```

## Summary

PHP PDO provides a secure and flexible interface for MySQL with support for prepared statements, named and positional parameters, transactions, and multiple fetch modes. Always use `PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION` for proper error handling and `PDO::ATTR_EMULATE_PREPARES => false` to use native prepared statements for better security.
