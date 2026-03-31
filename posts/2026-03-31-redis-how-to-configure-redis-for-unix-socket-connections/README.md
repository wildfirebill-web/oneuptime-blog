# How to Configure Redis for Unix Socket Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Unix Socket, Configuration, Performance, Linux

Description: Learn how to configure Redis to accept connections via a Unix domain socket for lower latency and better security when client and server run on the same host.

---

When your Redis client and server run on the same machine, using a Unix domain socket instead of a TCP connection reduces latency and avoids TCP overhead. This guide shows how to configure Redis for Unix socket connections and how to connect from common clients.

## Why Use Unix Sockets

Unix domain sockets use the filesystem instead of the network stack. Benefits include:
- Lower latency (no TCP handshake, no loopback interface)
- Filesystem-level access control via file permissions
- No port required, reducing attack surface

The trade-off is that Unix sockets only work when the client and server are on the same host.

## Configuring Redis for Unix Socket

Edit `/etc/redis/redis.conf`:

```text
# Disable TCP if you only need Unix socket connections
# port 0

# Or keep TCP enabled alongside Unix socket
port 6379

# Enable Unix socket
unixsocket /var/run/redis/redis.sock
unixsocketperm 770
```

The `unixsocketperm` value sets the file permission mode. `770` allows the owner and group to read and write. If your application user is in the `redis` group, `770` is a good choice.

Restart Redis to apply:

```bash
sudo systemctl restart redis
```

Verify the socket file exists:

```bash
ls -la /var/run/redis/redis.sock
```

## Connecting via the CLI

```bash
redis-cli -s /var/run/redis/redis.sock
redis-cli -s /var/run/redis/redis.sock PING
```

## Connecting from Python

```python
import redis

r = redis.Redis(
    unix_socket_path='/var/run/redis/redis.sock',
    decode_responses=True,
)
r.ping()
print(r.set('key', 'value'))
print(r.get('key'))
```

## Connecting from Node.js with ioredis

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  path: '/var/run/redis/redis.sock',
});

redis.set('key', 'value').then(() => {
  return redis.get('key');
}).then((val) => {
  console.log(val);
});
```

## Connecting from PHP with Predis

```php
<?php
require 'vendor/autoload.php';

$client = new Predis\Client([
    'scheme' => 'unix',
    'path'   => '/var/run/redis/redis.sock',
]);

$client->set('key', 'value');
echo $client->get('key');
```

## Connecting from Ruby

```ruby
require 'redis'

redis = Redis.new(path: '/var/run/redis/redis.sock')
redis.set('key', 'value')
puts redis.get('key')
```

## Setting File Permissions

Ensure the application user can access the socket file. Add the application user to the `redis` group:

```bash
sudo usermod -aG redis www-data   # For Apache/Nginx PHP
sudo usermod -aG redis myappuser
```

Then verify group membership:

```bash
id myappuser
```

## Benchmarking the Difference

```bash
# TCP
redis-benchmark -h 127.0.0.1 -p 6379 -n 100000 -c 50

# Unix socket
redis-benchmark -s /var/run/redis/redis.sock -n 100000 -c 50
```

## Summary

Configuring Redis for Unix socket connections requires setting `unixsocket` and `unixsocketperm` in `redis.conf`. This delivers lower latency than TCP for same-host deployments and provides filesystem-level access control. Ensure the application user has group access to the socket file, and use the `path` or `unix_socket_path` option in your Redis client library.
