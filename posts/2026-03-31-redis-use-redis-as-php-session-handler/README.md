# How to Use Redis as PHP Session Handler

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Php, Sessions, Session Handler, Cache

Description: Learn how to configure Redis as the PHP session handler to store user sessions centrally, enabling scalable multi-server PHP applications.

---

## Why Use Redis for PHP Sessions?

By default PHP stores sessions as files on disk. This works fine for a single server but breaks when you scale horizontally - sessions stored on server A are not available on server B. Using Redis as a centralized session store solves this problem and also provides:

- Fast in-memory reads and writes
- Automatic session expiration via TTL
- Shared session state across multiple PHP processes and servers

## Option 1 - Using the phpredis Extension

The phpredis C extension provides a built-in Redis session save handler. After installing the extension, update `php.ini`:

```ini
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379"
```

For password-protected Redis:

```ini
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379?auth=your-password"
```

Restart PHP-FPM or Apache after changing `php.ini`:

```bash
sudo systemctl restart php8.2-fpm
```

## Option 2 - Using Predis as a Custom Session Handler

If you prefer Predis (pure PHP, no C extension needed), implement the `SessionHandlerInterface`:

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

class RedisSessionHandler implements SessionHandlerInterface
{
    private Client $redis;
    private int $ttl;
    private string $prefix;

    public function __construct(Client $redis, int $ttl = 1440, string $prefix = 'session:')
    {
        $this->redis  = $redis;
        $this->ttl    = $ttl;
        $this->prefix = $prefix;
    }

    public function open(string $path, string $name): bool
    {
        return true;
    }

    public function close(): bool
    {
        return true;
    }

    public function read(string $id): string|false
    {
        $data = $this->redis->get($this->prefix . $id);
        return $data ?? '';
    }

    public function write(string $id, string $data): bool
    {
        $this->redis->setex($this->prefix . $id, $this->ttl, $data);
        return true;
    }

    public function destroy(string $id): bool
    {
        $this->redis->del($this->prefix . $id);
        return true;
    }

    public function gc(int $max_lifetime): int|false
    {
        // Redis handles TTL-based expiration automatically
        return 0;
    }
}
```

Register and use the handler:

```php
<?php
require 'vendor/autoload.php';

use Predis\Client;

$redis   = new Client(['host' => '127.0.0.1', 'port' => 6379]);
$handler = new RedisSessionHandler($redis, ttl: 3600);

session_set_save_handler($handler, true);
session_start();

$_SESSION['user_id']  = 42;
$_SESSION['username'] = 'alice';

echo "Session ID: " . session_id() . PHP_EOL;
echo "User: " . $_SESSION['username'] . PHP_EOL;
```

## Configuring Session Lifetime

Control how long sessions persist:

```php
<?php
// Set session cookie lifetime (client side)
ini_set('session.cookie_lifetime', 86400); // 1 day

// Set session garbage collection maxlifetime (server side)
ini_set('session.gc_maxlifetime', 86400);

session_start();
```

When using the Predis handler, pass the TTL as a constructor argument (in seconds):

```php
<?php
$handler = new RedisSessionHandler($redis, ttl: 86400); // 1 day
```

## Verifying Sessions Are Stored in Redis

After starting a session, verify the data is in Redis using redis-cli:

```bash
redis-cli keys "session:*"
redis-cli get "session:abc123def456"
```

You should see the serialized PHP session data.

## Multi-Server Configuration

For load-balanced environments, point all servers to the same Redis instance or cluster:

```ini
session.save_handler = redis
session.save_path = "tcp://redis.internal:6379?auth=secret&database=1"
```

All PHP servers will share session state through Redis, enabling sticky-session-free horizontal scaling.

## Session Security Best Practices

```php
<?php
// Regenerate session ID after login to prevent fixation attacks
session_start();
session_regenerate_id(true);
$_SESSION['user_id'] = $authenticatedUserId;

// Destroy session on logout
session_start();
session_destroy();
```

Use TLS for Redis connections in production:

```ini
session.save_path = "tls://redis.internal:6380?auth=secret"
```

## Summary

Redis is an excellent PHP session handler, providing shared session state across multiple servers and automatic expiration. You can use it with the phpredis extension by setting `session.save_handler = redis` in `php.ini`, or implement a custom `SessionHandlerInterface` with Predis for a pure-PHP solution. Both approaches centralize session storage and eliminate the single-server limitation of file-based sessions.
