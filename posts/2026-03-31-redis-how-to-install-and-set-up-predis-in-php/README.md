# How to Install and Set Up Predis in PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PHP, Predis, Installation, Setup, Composer

Description: Learn how to install and configure Predis, the pure-PHP Redis client library, using Composer, and verify your setup with basic connection tests.

---

## What Is Predis?

Predis is a flexible and feature-rich PHP client for Redis. Key features:

- Pure PHP implementation (no C extension required)
- Supports Redis Cluster and Sentinel
- Connection pooling and pipelining
- Pub/Sub support
- Easy to install via Composer

## Prerequisites

- PHP 7.2 or higher
- Composer installed
- Redis server running

## Installing via Composer

```bash
composer require predis/predis
```

Or add to your `composer.json`:

```json
{
  "require": {
    "predis/predis": "^2.0"
  }
}
```

Then install:

```bash
composer install
```

## Basic Connection

```php
<?php

require 'vendor/autoload.php';

use Predis\Client;

// Connect to local Redis
$redis = new Client();

// Or with explicit options
$redis = new Client([
    'scheme' => 'tcp',
    'host'   => '127.0.0.1',
    'port'   => 6379,
]);

// With password
$redis = new Client([
    'host'     => '127.0.0.1',
    'port'     => 6379,
    'password' => 'yourpassword',
    'database' => 0,
]);

// Test the connection
$pong = $redis->ping();
echo $pong; // PONG
```

## Connecting with a URL

```php
<?php

require 'vendor/autoload.php';

use Predis\Client;

// Simple URL
$redis = new Client('redis://localhost:6379');

// With password
$redis = new Client('redis://:yourpassword@localhost:6379/0');

// TLS
$redis = new Client('rediss://redis.example.com:6380');
```

## Connection with SSL/TLS

```php
<?php

use Predis\Client;

$redis = new Client([
    'scheme' => 'tls',
    'host'   => 'redis.example.com',
    'port'   => 6380,
    'ssl'    => [
        'cafile'      => '/path/to/ca.crt',
        'local_cert'  => '/path/to/client.crt',
        'local_pk'    => '/path/to/client.key',
        'verify_peer' => true,
    ],
]);
```

## Configuring Timeouts

```php
<?php

use Predis\Client;

$redis = new Client([
    'host'                  => '127.0.0.1',
    'port'                  => 6379,
    'timeout'               => 2.0,    // Connection timeout (seconds)
    'read_write_timeout'    => 5.0,    // Command timeout (seconds)
    'persistent'            => false,  // Use persistent connections
]);
```

## Verifying the Installation

```php
<?php

require 'vendor/autoload.php';

use Predis\Client;
use Predis\Connection\ConnectionException;

function verifyRedisConnection(string $host = '127.0.0.1', int $port = 6379): void {
    try {
        $redis = new Client(['host' => $host, 'port' => $port]);

        // Test basic operations
        $pong = $redis->ping();
        echo "PING: {$pong}" . PHP_EOL;

        $redis->set('test:setup', 'hello');
        $value = $redis->get('test:setup');
        echo "GET test:setup: {$value}" . PHP_EOL;

        $redis->del('test:setup');

        // Get server info
        $info = $redis->info('server');
        echo "Redis version: " . $info['Server']['redis_version'] . PHP_EOL;
        echo "Connected clients: " . $redis->info('clients')['Clients']['connected_clients'] . PHP_EOL;

        echo "Predis setup successful!" . PHP_EOL;

    } catch (ConnectionException $e) {
        echo "Connection failed: " . $e->getMessage() . PHP_EOL;
        exit(1);
    } catch (\Exception $e) {
        echo "Error: " . $e->getMessage() . PHP_EOL;
        exit(1);
    }
}

verifyRedisConnection();
```

## Configuration in a PHP Application

Create a reusable Redis configuration class:

```php
<?php

require 'vendor/autoload.php';

use Predis\Client;

class RedisConfig {
    private static ?Client $instance = null;

    private static array $defaultOptions = [
        'host'               => '127.0.0.1',
        'port'               => 6379,
        'password'           => null,
        'database'           => 0,
        'timeout'            => 2.0,
        'read_write_timeout' => 5.0,
    ];

    public static function getInstance(array $options = []): Client {
        if (self::$instance === null) {
            $config = array_merge(self::$defaultOptions, $options);

            // Remove null values
            $config = array_filter($config, fn($v) => $v !== null);

            self::$instance = new Client($config);
        }
        return self::$instance;
    }
}

// Usage
$redis = RedisConfig::getInstance([
    'host'     => getenv('REDIS_HOST') ?: '127.0.0.1',
    'port'     => (int) (getenv('REDIS_PORT') ?: 6379),
    'password' => getenv('REDIS_PASSWORD') ?: null,
]);

echo $redis->ping(); // PONG
```

## Using with Laravel

Laravel has built-in Redis support using Predis. In `config/database.php`:

```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'predis'),

    'default' => [
        'host'     => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port'     => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],
],
```

Install Predis for Laravel:

```bash
composer require predis/predis
```

Use in Laravel code:

```php
use Illuminate\Support\Facades\Redis;

Redis::set('key', 'value');
$value = Redis::get('key');
```

## Summary

Installing Predis in PHP requires just `composer require predis/predis` and including the Composer autoloader. Create a `Client` instance with host, port, and password configuration, test the connection with `ping()`, and wrap initialization in a singleton pattern for application-wide reuse. Predis integrates natively with Laravel and other PHP frameworks, making it the standard choice for PHP Redis clients when a C extension is not available.
