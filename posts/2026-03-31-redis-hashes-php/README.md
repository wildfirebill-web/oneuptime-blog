# How to Use Redis Hashes in PHP

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PHP, Hash, Data Structure, phpredis

Description: Learn how to use Redis Hashes in PHP with phpredis and Predis to store and retrieve structured objects with field-level operations.

---

Redis Hashes store key-value pairs within a single Redis key, making them ideal for representing objects like users, products, and sessions without deserializing an entire blob just to update one field.

## Basic Hash Operations with phpredis

```php
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

// Set individual fields
$redis->hSet('user:42', 'name', 'Alice');
$redis->hSet('user:42', 'email', 'alice@example.com');
$redis->hSet('user:42', 'age', '30');

// Set multiple fields at once
$redis->hMSet('user:42', [
    'name'  => 'Alice',
    'email' => 'alice@example.com',
    'age'   => '30',
    'role'  => 'admin',
]);
```

## Reading Hash Fields

```php
// Get a single field
$name = $redis->hGet('user:42', 'name');
echo $name; // Alice

// Get multiple specific fields
$data = $redis->hMGet('user:42', ['name', 'email']);
print_r($data);

// Get all fields
$user = $redis->hGetAll('user:42');
print_r($user);
// ['name' => 'Alice', 'email' => 'alice@example.com', ...]
```

## Checking and Deleting Fields

```php
// Check if a field exists
if ($redis->hExists('user:42', 'role')) {
    echo "Has role field\n";
}

// Delete a field
$redis->hDel('user:42', 'role');

// Count number of fields
$count = $redis->hLen('user:42');
echo "Fields: $count\n"; // 3
```

## Numeric Field Operations

```php
// Increment a numeric field
$redis->hSet('product:1', 'stock', '100');
$newStock = $redis->hIncrBy('product:1', 'stock', -5);
echo $newStock; // 95

// Float increment
$redis->hIncrByFloat('product:1', 'price', 1.50);
```

## Getting Keys and Values Separately

```php
$keys   = $redis->hKeys('user:42');   // ['name', 'email', 'age']
$values = $redis->hVals('user:42');   // ['Alice', 'alice@example.com', '30']
```

## Hashes with Predis

```php
use Predis\Client;

$client = new Client();

$client->hmset('session:xyz', [
    'user_id'    => '7',
    'ip'         => '192.168.1.1',
    'created_at' => (string) time(),
]);

$session = $client->hgetall('session:xyz');
echo $session['user_id']; // 7
```

## Practical Example - User Profile Cache

```php
function saveUserProfile(Redis $redis, int $userId, array $profile): void
{
    $key = "user:profile:{$userId}";
    $redis->hMSet($key, $profile);
    $redis->expire($key, 3600);
}

function getUserProfile(Redis $redis, int $userId): array
{
    return $redis->hGetAll("user:profile:{$userId}");
}

saveUserProfile($redis, 1, [
    'name'   => 'Bob',
    'email'  => 'bob@example.com',
    'plan'   => 'pro',
]);

$profile = getUserProfile($redis, 1);
echo $profile['plan']; // pro
```

## Memory Efficiency

Redis uses a compact encoding (ziplist/listpack) for hashes with fewer than 128 fields and small values, making them very memory-efficient for object storage compared to storing serialized strings.

## Summary

Redis Hashes in PHP are the right tool for structured object storage when you need field-level reads and writes. Both phpredis and Predis provide the full Hash command set. Use hashes for user profiles, sessions, product records, and any data you want to update field by field without loading the entire object.
