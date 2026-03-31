# How to Connect to Redis from PHP with Predis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PHP, Predis, Connection, Data Types

Description: Learn how to connect to Redis from PHP using Predis and perform operations on strings, hashes, lists, sets, and sorted sets with practical examples.

---

## Connecting to Redis

```php
<?php

require 'vendor/autoload.php';

use Predis\Client;
use Predis\Connection\ConnectionException;

function createRedisClient(array $options = []): Client {
    $defaults = [
        'scheme'             => 'tcp',
        'host'               => getenv('REDIS_HOST') ?: '127.0.0.1',
        'port'               => (int) (getenv('REDIS_PORT') ?: 6379),
        'password'           => getenv('REDIS_PASSWORD') ?: null,
        'database'           => 0,
        'timeout'            => 2.0,
        'read_write_timeout' => 5.0,
    ];

    $config = array_filter(array_merge($defaults, $options), fn($v) => $v !== null);

    return new Client($config);
}

try {
    $redis = createRedisClient();
    echo "Connected: " . $redis->ping() . PHP_EOL;
} catch (ConnectionException $e) {
    die("Cannot connect to Redis: " . $e->getMessage());
}
```

## String Operations

```php
<?php

$redis = new Client();

// Basic set/get
$redis->set('greeting', 'Hello, Redis!');
$value = $redis->get('greeting');
echo $value . PHP_EOL; // Hello, Redis!

// Set with expiry (seconds)
$redis->setex('session:token:abc', 3600, 'user_data_here');

// Set only if not exists
$result = $redis->setnx('unique:key', 'value');
echo $result ? "Set" : "Already exists";

// Get and set atomically
$old = $redis->getset('key', 'new_value');

// Increment/Decrement
$redis->set('page:views', 0);
$redis->incr('page:views');
$redis->incrby('page:views', 5);
echo $redis->get('page:views') . PHP_EOL; // 6

// String length
$redis->set('bio', 'Redis developer');
echo $redis->strlen('bio') . PHP_EOL; // 15

// Append
$redis->append('bio', ' and architect');

// Check existence and TTL
echo $redis->exists('greeting') . PHP_EOL; // 1
echo $redis->ttl('session:token:abc') . PHP_EOL; // remaining seconds

// Delete
$redis->del('greeting');
```

## Hash Operations

```php
<?php

$redis = new Client();

// Set a single field
$redis->hset('user:1001', 'name', 'Alice');

// Set multiple fields
$redis->hmset('user:1001', [
    'email'  => 'alice@example.com',
    'age'    => '30',
    'role'   => 'admin',
    'active' => '1',
]);

// Get a single field
$name = $redis->hget('user:1001', 'name');
echo $name . PHP_EOL;

// Get multiple fields
[$email, $role] = $redis->hmget('user:1001', ['email', 'role']);
echo "$email - $role" . PHP_EOL;

// Get all fields
$user = $redis->hgetall('user:1001');
print_r($user);

// Get all keys / values
$keys = $redis->hkeys('user:1001');
$vals = $redis->hvals('user:1001');

// Count fields
echo $redis->hlen('user:1001') . PHP_EOL;

// Delete a field
$redis->hdel('user:1001', 'age');

// Check field existence
echo $redis->hexists('user:1001', 'email') . PHP_EOL; // 1

// Increment a hash field
$redis->hset('user:1001', 'login_count', 0);
$redis->hincrby('user:1001', 'login_count', 1);
```

## List Operations

```php
<?php

$redis = new Client();

// Push to end of list (right)
$redis->rpush('tasks', 'task:1', 'task:2', 'task:3');

// Push to beginning (left)
$redis->lpush('tasks', 'urgent:task');

// Pop from front (FIFO queue)
$task = $redis->lpop('tasks');
echo "Processing: $task" . PHP_EOL;

// Pop from end
$last = $redis->rpop('tasks');

// Get range
$all = $redis->lrange('tasks', 0, -1);

// Get length
echo $redis->llen('tasks') . PHP_EOL;

// Get by index
$first = $redis->lindex('tasks', 0);

// Trim to fixed size
$redis->ltrim('recent:activity', 0, 99); // Keep latest 100

// Blocking pop (waits up to 5 seconds)
$result = $redis->blpop(['queue:jobs'], 5);
if ($result !== null) {
    [$key, $value] = $result;
    echo "Got job from $key: $value" . PHP_EOL;
}
```

## Set Operations

```php
<?php

$redis = new Client();

// Add members
$redis->sadd('user:1001:interests', 'redis', 'php', 'databases', 'caching');

// Check membership
echo $redis->sismember('user:1001:interests', 'redis') . PHP_EOL; // 1

// Get all members
$interests = $redis->smembers('user:1001:interests');

// Count
echo $redis->scard('user:1001:interests') . PHP_EOL;

// Remove
$redis->srem('user:1001:interests', 'caching');

// Set operations
$redis->sadd('user:1002:interests', 'redis', 'mysql', 'python');

$common = $redis->sinter('user:1001:interests', 'user:1002:interests');
$all    = $redis->sunion('user:1001:interests', 'user:1002:interests');
$unique = $redis->sdiff('user:1001:interests', 'user:1002:interests');

echo "Common interests: " . implode(', ', $common) . PHP_EOL;
```

## Sorted Set Operations

```php
<?php

$redis = new Client();

// Add members with scores
$redis->zadd('leaderboard', ['alice' => 1500, 'bob' => 2300, 'carol' => 1800]);

// Get rank (highest score = rank 0)
$rank = $redis->zrevrank('leaderboard', 'alice');
echo "Alice rank: " . ($rank + 1) . PHP_EOL;

// Get score
$score = $redis->zscore('leaderboard', 'bob');
echo "Bob score: $score" . PHP_EOL;

// Increment score
$newScore = $redis->zincrby('leaderboard', 200, 'alice');

// Get top 10 with scores
$top10 = $redis->zrevrange('leaderboard', 0, 9, ['WITHSCORES' => true]);
foreach ($top10 as $player => $score) {
    echo "$player: $score" . PHP_EOL;
}

// Get by score range
$midRange = $redis->zrangebyscore('leaderboard', 1500, 2000, ['WITHSCORES' => true]);

// Count in range
echo $redis->zcount('leaderboard', 1000, '+inf') . PHP_EOL;
```

## Summary

Predis provides a clean, PHP-native API for all Redis data types including strings, hashes, lists, sets, and sorted sets. Operations map closely to Redis commands (`hset`, `hmget`, `sadd`, `zadd`, etc.), making the Redis documentation directly applicable. Use `lrange` with `0, -1` to get all list elements, `hgetall` for complete hash objects, and `zrevrange` with `WITHSCORES` for leaderboard data. Always wrap connections in try/catch for `ConnectionException` to handle Redis unavailability gracefully.
