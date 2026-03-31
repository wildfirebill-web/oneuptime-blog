# How to Install and Set Up Predis in PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Php, Predis, Cache, Client Library

Description: Learn how to install and configure Predis, the pure PHP Redis client, using Composer and connect it to a Redis server in your PHP project.

---

## What is Predis?

Predis is a pure PHP client library for Redis. Unlike the phpredis extension (which requires compiling a C extension), Predis works entirely in PHP, making it easy to install via Composer without any system-level dependencies. It supports all Redis commands, pipelines, transactions, and pub/sub.

## Prerequisites

- PHP 7.2 or higher
- Composer installed
- A running Redis server (local or remote)

## Installing Predis via Composer

Run the following command in your project root:

```bash
composer require predis/predis
```

Composer downloads the package and adds it to `composer.json`:

```json
{
    "require": {
        "predis/predis": "^2.0"
    }
}
```

Include the Composer autoloader in your PHP script:

```php
<?php
require 'vendor/autoload.php';
```

## Connecting to Redis

### Default Connection (localhost:6379)

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

$redis = new Client();

$redis->set('greeting', 'Hello from Predis');
echo $redis->get('greeting'); // Hello from Predis
```

### Custom Host and Port

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

$redis = new Client([
    'scheme' => 'tcp',
    'host'   => '127.0.0.1',
    'port'   => 6379,
]);
```

### Connecting with a Password

```php
<?php
use Predis\Client;

$redis = new Client([
    'scheme'   => 'tcp',
    'host'     => '127.0.0.1',
    'port'     => 6379,
    'password' => 'your-redis-password',
]);
```

### Connecting via a URL String

```php
<?php
use Predis\Client;

$redis = new Client('redis://:your-redis-password@127.0.0.1:6379/0');
```

The URL format is: `redis://[password@]host[:port][/db-index]`

## Selecting a Database

Redis has 16 databases by default (0-15). Select a specific one:

```php
<?php
use Predis\Client;

$redis = new Client([
    'host'     => '127.0.0.1',
    'port'     => 6379,
    'database' => 1,
]);
```

## Testing the Connection

Use the PING command to verify the connection is healthy:

```php
<?php
use Predis\Client;

$redis = new Client();

try {
    $response = $redis->ping();
    echo "Connection status: " . $response->getPayload() . PHP_EOL; // PONG
} catch (\Predis\Connection\ConnectionException $e) {
    echo "Could not connect to Redis: " . $e->getMessage() . PHP_EOL;
}
```

## Basic Key Operations

```php
<?php
use Predis\Client;

$redis = new Client();

// Set a key with expiration
$redis->setex('session:abc123', 3600, json_encode(['user_id' => 42]));

// Get value
$value = $redis->get('session:abc123');
echo $value; // {"user_id":42}

// Check TTL
$ttl = $redis->ttl('session:abc123');
echo "TTL: $ttl seconds" . PHP_EOL;

// Delete a key
$redis->del('session:abc123');

// Check if key exists
$exists = $redis->exists('session:abc123');
echo $exists ? "Key exists" : "Key not found";
```

## Connection Pooling and Persistent Connections

Predis supports persistent connections to reduce connection overhead:

```php
<?php
use Predis\Client;

$redis = new Client([
    'host'       => '127.0.0.1',
    'port'       => 6379,
    'persistent' => true,
]);
```

## Error Handling

```php
<?php
use Predis\Client;
use Predis\Connection\ConnectionException;
use Predis\Response\ServerException;

$redis = new Client();

try {
    $redis->set('key', 'value');
    $result = $redis->get('key');
    echo $result . PHP_EOL;
} catch (ConnectionException $e) {
    // Redis server is unreachable
    error_log("Redis connection error: " . $e->getMessage());
} catch (ServerException $e) {
    // Redis returned an error response
    error_log("Redis server error: " . $e->getMessage());
}
```

## Verifying the Setup

Run a quick verification script:

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

$redis = new Client();

// Write and read back
$redis->set('test:setup', 'predis-ok');
$val = $redis->get('test:setup');

if ($val === 'predis-ok') {
    echo "Predis is set up correctly!" . PHP_EOL;
} else {
    echo "Something went wrong." . PHP_EOL;
}

// Cleanup
$redis->del('test:setup');
```

## Summary

Predis is the easiest way to add Redis support to a PHP project - install it with `composer require predis/predis`, require the autoloader, and instantiate `Predis\Client` with your server details. It supports all Redis commands natively and handles connection errors through PHP exceptions, making it straightforward to integrate into any PHP application without system-level dependencies.
